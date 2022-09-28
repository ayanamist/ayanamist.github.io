---
layout: post
title: k8s里的一些小陷阱
---

# client-go informer中的内存泄露

这个已经报告给了社区 https://github.com/kubernetes/kubernetes/issues/111370

这其实是个go语言的坑，当一个slice的元素不是指针，又直接对某个元素取指针的时候，指针引用的是整个slice底层array，而不是单一元素。

官方要解决的话，只能对每个元素用反射做一次浅拷贝了；我自己是简单fork了一下，允许替换ThreadSafeStore的实现：

```diff
diff --git a/tools/cache/thread_safe_store.go b/tools/cache/thread_safe_store.go
index 71d909d4..cbc903ec 100644
--- a/tools/cache/thread_safe_store.go
+++ b/tools/cache/thread_safe_store.go
@@ -327,7 +327,9 @@ func (c *threadSafeMap) Resync() error {
 }
 
 // NewThreadSafeStore creates a new instance of ThreadSafeStore.
-func NewThreadSafeStore(indexers Indexers, indices Indices) ThreadSafeStore {
+var NewThreadSafeStore = DefaultNewThreadSafeStore
+
+func DefaultNewThreadSafeStore(indexers Indexers, indices Indices) ThreadSafeStore {
        return &threadSafeMap{
                items:    map[string]interface{}{},
                indexers: indexers,
```

然后自己实现了一个，对每个元素做了一次shallow copy来打断引用关系，当然最好先判断下类型，然后用type assert转换一下，不然反射要慢50倍

```go
func (t *myThreadSafeStore) copyObj(obj interface{}) interface{} {
	if pod, ok := obj.(*corev1.Pod); ok {
		pod2 := *pod
		obj = &pod2
	} else if node, ok := obj.(*corev1.Node); ok {
		node2 := *node
		obj = &node2
	} else if value := reflect.ValueOf(obj); value.Kind() == reflect.Ptr {
		// 尽量不用反射，要慢50倍
		value = value.Elem()
		typ := value.Type()
		value2 := reflect.New(typ)
		value2.Elem().Set(value)
		obj = value2.Interface()
	}
	return obj
}
```

# informer默认的list会造成瞬时内存占用彪高

informer中使用的reflector是先list获得一个resourceVersion再用这个rv去做watch，而list的方法则是

https://github.com/kubernetes/kubernetes/blob/release-1.25/staging/src/k8s.io/client-go/tools/cache/reflector.go#L357

而list用的resourceVersion来自relistResourceVersion()

https://github.com/kubernetes/kubernetes/blob/release-1.25/staging/src/k8s.io/client-go/tools/cache/reflector.go#L582

可以看到第一次用的是"0"

```go
	if r.lastSyncResourceVersion == "" {
		// For performance reasons, initial list performed by reflector uses "0" as resource version to allow it to
		// be served from the watch cache if it is enabled.
		return "0"
	}
```

而list resourceVersion="0"时是全量返回所有apiserver已经缓存好的资源

https://github.com/kubernetes/kubernetes/blob/release-1.25/staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go#L679

https://github.com/kubernetes/kubernetes/blob/release-1.25/staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go#L618

https://github.com/kubernetes/kubernetes/blob/release-1.25/staging/src/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go#L471

而client-go对于list的处理都是全量读出一个[]byte然后再丢给decoder去解析，这一瞬间会有两份全量pod的内存占用

https://github.com/kubernetes/kubernetes/blob/release-1.25/staging/src/k8s.io/client-go/kubernetes/typed/core/v1/pod.go#L88

https://github.com/kubernetes/kubernetes/blob/release-1.25/staging/src/k8s.io/client-go/rest/request.go#L1294

# watch resourceVersion="0" 有可能永远不会成功

按上面list的坑，肯定就想着，是不是可以按 https://kubernetes.io/docs/reference/using-api/api-concepts/#semantics-for-watch 这里 resourceVersion="0" 的行为，流式获取所有pod呢

> To establish initial state, the watch begins with synthetic "Added" events for all resource instances that exist at the starting resource version.

但抱歉，apiserver这里的实现，在建立watch的同时，不等initEvents发送完，就开始按当前时间戳接收新的event了

1. 确定chanSize
    https://github.com/kubernetes/kubernetes/blob/release-1.25/staging/src/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go#L608
    
    可以看到最大也就是1000
    https://github.com/kubernetes/kubernetes/blob/release-1.25/staging/src/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go#L589
2. 获取存量的event
    https://github.com/kubernetes/kubernetes/blob/release-1.25/staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go#L520

    resourceVersion="0"时返回存量全部数据
    https://github.com/kubernetes/kubernetes/blob/a7d69e8884a40b09cffd89cc5cbc085bf0ac97c2/staging/src/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go#L664
3. 先从`cacheInterval`发存量的event
    https://github.com/kubernetes/kubernetes/blob/release-1.25/staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go#L1405

    然后再消费`c.input`
    https://github.com/kubernetes/kubernetes/blob/release-1.25/staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go#L1481
4. 但消费`cacheInterval`的同时，Cacher全局已经开始通过`nonblockingAdd`和`add`往`c.input`里放event了
    https://github.com/kubernetes/kubernetes/blob/release-1.25/staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go#L880
    https://github.com/kubernetes/kubernetes/blob/release-1.25/staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go#L1239

    https://github.com/kubernetes/kubernetes/blob/release-1.25/staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go#L899
    https://github.com/kubernetes/kubernetes/blob/release-1.25/staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go#L1249

    可以注意到`add`里面如果`c.input`放慢了，是不会无限等待的，而是等一小会就把这个watch关闭了

    这个超时的计算逻辑有点小复杂，`dispatchTimeoutBudget`是一个timeBudget对象
    https://github.com/kubernetes/kubernetes/blob/release-1.25/staging/src/k8s.io/apiserver/pkg/storage/cacher/time_budget.go#L31
    maxBudget为100毫秒，refreshPerSecond为50毫秒，类似漏桶模型，每秒最多可用的超时为100毫秒，每秒恢复50毫秒。可以简单的理解为如果100毫秒内放不进`c.input`就把watch关了。

问题就来了，需要用这个方法来获取全量pod的都是大集群，大集群往往event也很多，我这里稍大的集群每秒都有50条event，最大的每秒500条event，就意味着在20秒甚至2秒内不能消费完initEvents，watch就会被apiserver关掉，而我这里什么都不干的空watch每秒也只能处理1200条event，所以几乎永远不可能获取完全initEvent，而resourceVersion="0"的watch是不能断点续传的……所以就永远无法成功了。