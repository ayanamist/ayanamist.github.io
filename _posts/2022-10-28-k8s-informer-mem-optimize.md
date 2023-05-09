---
layout: post
title: k8s client-go内存优化
---

[client-go](https://pkg.go.dev/k8s.io/client-go) 是每个用go语言连接k8s集群的必备之物。在集团大规模场景下，也暴露出它在内存使用上的一些问题。

# 优先使用Protobuf而不是JSON

首先容易踩的坑，是默认client-go是使用json而不是protobuf的。

这里先写一个简单的小程序，用ResourceVersion=0全量拉取所有Pod，默认参数下

![](/img/2022-10-28-16-24-24.png)

接着加上protobuf：使用 [metadata.ConfigFor](https://pkg.go.dev/k8s.io/client-go/metadata@v0.20.15#ConfigFor) 进行配置

![](/img/2022-10-28-16-25-52.png)

这个函数就是配置了优先使用protobuf编码

![](/img/2022-10-28-16-26-32.png)

再重新拉取全量Pod

![](/img/2022-10-28-16-27-18.png)

可以发现，apiserver的响应时间，从35秒下降到13秒，拉取到的数据量也从3.4G下降到2.7G，反序列化的耗时从58秒下降到14秒，效果是非常显著的。

这是裸用client-go最容易遗漏的地方，加一行就可以直接有巨大收益

# 流式list

大部分用户都会直接或间接使用client-go中的[informer](https://pkg.go.dev/k8s.io/client-go@v0.25.3/tools/cache#NewSharedIndexInformer)来访问k8s集群的资源，间接的方式主要包括：用[informers.SharedInformerFactory](https://pkg.go.dev/k8s.io/client-go@v0.25.3/informers#NewSharedInformerFactory)，底层依然是informer，或者[controller-runtime](https://pkg.go.dev/sigs.k8s.io/controller-runtime)底层同样是一个informer

而informer最关键的就是传入的[cache.ListerWatcher](https://pkg.go.dev/k8s.io/client-go@v0.25.3/tools/cache#ListerWatcher)

![](/img/2022-10-28-16-52-19.png)

通读一遍代码或者网上找一下文章可以知道，整个informer就是通过启动的时候调用一下List获取全量数据，然后用List得到的resourceVersion启动Watch获取增量数据，将获得的的所有数据都缓存在内存中来降低对apiserver的请求压力的。

而启动的这次List用的resourceVersion一定是0

![](/img/2022-10-28-16-55-11.png)

而resourceVersion=0时的语义，就是无视limit分页，一次性从apiserver的内存cache中获取全量数据（label selector和field selector还是会遵守的）

但我们看`list`的实现，会发现他丫的居然是把整个请求先全部读取到内存中`r.body`再做一次全量反序列化。

![](/img/2022-10-28-16-57-04.png)
![](/img/2022-10-28-16-57-30.png)

因此如果在一个超大集群，这内存占用就会蹭蹭蹭的暴涨，以asi的压测集群 asi_zjk_perf_test01 为例，里面包含了 57w 个Pod（都是伪造的），启动一个informer后

![](/img/2022-10-28-17-05-54.png)

光list完成，就花了快4分钟，同时截图中黄框里gctrace的日志出现了接近200ms的[STW停顿](https://pkg.go.dev/runtime#hdr-Environment_Variables:~:text=wall%2Dclock/CPU%20times%20for%20the%20phases%20of%20the%20GC)，最终RSS达到了惊人的74G，强制触发一次`debug.FreeOSMemory()`后释放了一半内存（37G），可想而知，为了这启动的瞬时内存翻倍，就必须常态的将内存指定到74G以上，并且运行过程中还可能因为watch断开等原因，再触发一次list，那就会在已经用了37G的基础上再叠加一个74G的临时分配，111G的内存占用浪费严重，也容易触发系统oom killer，哪怕容器会自动重启，恢复也要4分钟，可用性影响很大。

那有没有办法这里改成流式的呢，至少不要因为全量拿到数据后再做全量反序列化，这样反序列化过程中实际是占了两份内存的。

流式获取数据，client-go为watch场景留了一个[Stream](https://pkg.go.dev/k8s.io/client-go/rest@v0.20.15#Request.Stream)接口，除了取不到Content-Type，其实也够用，因为Header里也没有什么有用的信息了。而且Content-Type也不重要，有其他判断方法，下面会说。

接着就要看protobuf解析的逻辑了，解析逻辑在[protobuf.Serializer#Decode](https://github.com/kubernetes/apimachinery/blob/v0.20.15/pkg/runtime/serializer/protobuf/protobuf.go#L99)方法里：

1. 判断前四个字节是不是`k8s\x0`，这是protobuf编码的magic code，因此也可以用这个来替代content-type的检查
2. 将全量数据反序列化成[runtime.Unknown](https://pkg.go.dev/k8s.io/apimachinery/pkg/runtime@v0.20.15#Unknown)，获取GVK信息
3. 将Unknown.Raw通过`proto.Unmarshal`反序列化到最终的对象，调用的是对象自己的[Unmarshal](https://pkg.go.dev/k8s.io/api/core/v1@v0.20.15#PodList.Unmarshal)方法

所以如果要支持流式解析，就必须支持两个对象的嵌套流式解析，在Unknown对象流式解析的基础上，不等Raw字段全部解析完，将Raw再作为一个流喂给PodList对象的流式解析。

这里通读一下protobuf的`Unmarshal`实现，可以发现`iNdEx`是一直向前的，同时只有两个操作，一个是从`dAtA`里取一个字节判断字段类型，一个是取连续的若干字节作为字段内容，再去调用这个字段自己的`Unmarshal`

我们只需要到Pod粒度即可，不需要像sax风格的解析器那样支持每个字段的流式解析，同时考虑到k8s 几乎所有的XXXList对象的格式都是一样的，包含metav1.ListMeta和Items两个字段（metav1.TypeMeta在protobuf字节流中不存在），因此可以改造出一个通用的解析器。

同时为了方便最大程度的利用已有的protobuf解析代码，又实现了一个`streamBuffer`对象来模拟`[]byte`的操作

整块代码实现可以见 https://github.com/ayanamist/k8s-utils/blob/3d5e35f70386d8dceebdc070cadecb9be0c518bd/pkg/streamlister

同时有个能跑通的实例说明怎么使用 https://github.com/ayanamist/k8s-utils/blob/3d5e35f70386d8dceebdc070cadecb9be0c518bd/cmd/streamlister/main.go

配合informer的使用，可以用如下代码：

```go
func listPod(ctx context.Context, client rest.Interface, namespace string, options metav1.ListOptions) (runtime.Object, error) {
	podList := &corev1.PodList{}
	err := streamlister.StreamList(ctx, client, "pods", namespace, options, streamlister.Param{
		ObjectFactoryFunc: func() proto.Unmarshaler {
			return &corev1.Pod{}
		},
		OnListMetaFunc: func(meta *metav1.ListMeta) {
			podList.ListMeta = *meta
		},
		OnObjectFunc: func(o proto.Unmarshaler) {
			podList.Items = append(podList.Items, *o.(*corev1.Pod))
		},
	})
	return podList, err
}

func startInformerFactory(ctx context.Context, client *kubernetes.Clientset) informers.SharedInformerFactory {
	informerFactory := informers.NewSharedInformerFactory(client, time.Hour)

	informerFactory.InformerFor(&corev1.Pod{}, func(client kubernetes.Interface, resyncDur time.Duration) cache.SharedIndexInformer {
		const namespace = ""
		return cache.NewSharedIndexInformer(
			&cache.ListWatch{
				ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
					return listPod(ctx, client.CoreV1().RESTClient(), namespace, options)
				},
				WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
					return client.CoreV1().Pods(namespace).Watch(ctx, options)
				},
			},
			&corev1.Pod{},
			resyncDur,
			cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc},
		)
	})

	informerFactory.Start(ctx.Done())
	informerFactory.WaitForCacheSync(ctx.Done())

	return informerFactory
}
```

由于informerFactory只会对同一个对象类型保存一个informer

![](/img/2022-10-28-17-53-41.png)

因此后续依旧可以`informerFactory.Core().V1().Pods().Lister()`获取一个`lister`对象正常使用

# `[]v1.Pod`的坑

考虑下informer的工作原理，后续是不断有watch到的对象最新状态会放到内部的store里，因此我们可以模拟下，当「几乎」所有Pod都被替换的时候，会发生什么?

按上述代码组装出`informerFactory`后，做一次`debug.FreeOSMemory`，然后将内部除了第二个以外的其他所有元素，全部替换成空Pod信息

![](/img/2022-10-28-17-58-04.png)

预期是内存会降低，因为每个Pod.Spec占用的空间都没了，但实际情况却是：

![](/img/2022-10-28-18-02-34.png)

没错，内存占用还上升了，而且GC也释放不掉！这里是把Spec和Status都清空了，如果是真实的生产环境，内存实际上是有可能翻倍的。也就是说，前面流式解析好不容易省下来的内存，最后还是会因为这个问题重新变成两倍的状态。

这个问题的原因要解释，必须先过一遍informer对list的处理流程：informer内部的reflector的list过程中会使用[meta.ExtractList](https://github.com/kubernetes/apimachinery/blob/v0.20.15/pkg/api/meta/help.go#L169)取出Items里的所有元素并转换成指针形式，相当于将`[]v1.Pod`转换成`[]*v1.Pod`，再通过[syncWith](https://github.com/kubernetes/client-go/blob/v0.20.15/tools/cache/reflector.go#L445)放到内部的store中。

问题就出在`meta.ExtractList`转换过程中踩到了go语言的坑。go里`pod := &items[1]`这种方式取出来的指针，实际引用了整个`items`对象，因此上面所说的将`[]v1.Pod`转换成`[]*v1.Pod`，`[]*v1.Pod`里的每个对象，都依然持有了对原本的`[]v1.Pod`这个巨大slice的引用，即使后面放到store里也不例外，因此后续watch不断收到各个pod新的事件进行覆盖，但只要还有一个pod没有被watch事件覆盖，那原本list得到的巨大list就依然存活无法被gc而占用着内存，而这种在超大规模集群里是很容易发生的（例如某个机器宕机，上面的pod都不会有kubelet的定期上报产生event了）

这个问题查出来，多亏了 mosn 团队 [doujiang24](https://github.com/doujiang24) 的大力帮助，使用了mosn修改版的 [viewcore](https://github.com/mosn/debug/tree/mosn/cmd/viewcore) 同时该问题也已经反馈给官方并附了一个最小复现方法 https://github.com/kubernetes/kubernetes/issues/111370 但老实说，在不改接口的情况下client-go自身是无解的

但client-go无解不代表我们无解，实际上`meta.ExtractList`留了一个口子，不仅可以对`[]v1.Pod`进行操作，同时也可以对`[]*v1.Pod`也进行操作，所以如果提前能在list里就返回`[]*v1.Pod`就可以绕过这个问题：我们新建一个`PodPtrList`，Items使用`[]*v1.Pod`然后修改上面的`listPod`方法即可

```go
type PodPtrList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`
	Items           []*corev1.Pod
}

func (m *PodPtrList) DeepCopyObject() runtime.Object {
	c := &PodPtrList{}
	c.TypeMeta = m.TypeMeta
	m.ListMeta.DeepCopyInto(&c.ListMeta)
	c.Items = make([]*corev1.Pod, len(m.Items))
	for i, p := range m.Items {
		c.Items[i] = p.DeepCopy()
	}
	return c
}

func listPod(ctx context.Context, client rest.Interface, namespace string, options metav1.ListOptions) (runtime.Object, error) {
	podList := &PodPtrList{}
	err := streamlister.StreamList(ctx, client, "pods", namespace, options, streamlister.Param{
		ObjectFactoryFunc: func() proto.Unmarshaler {
			return &corev1.Pod{}
		},
		OnListMetaFunc: func(meta *metav1.ListMeta) {
			podList.ListMeta = *meta
		},
		OnObjectFunc: func(o proto.Unmarshaler) {
			podList.Items = append(podList.Items, o.(*corev1.Pod))
		},
	})
	return podList, err
}
```

这样在源头就打断了引用关系。代码替换进去，重新测试下，可以看到由于GC次数变少了，GC时间也变短了，整体list的耗时也大幅度缩短；同时list完成后的内存占用也从前面的74G下降到40G，并且模拟update后内存是进一步下降的。

![](/img/2022-10-28-20-46-24.png)

# GC goal抬高的风险

注意图中GC goal的变化了，可以看到在第一个黄框旁边的`53125 MB goal`表示heap达到这个占用之前都不会再因为heap上升而触发GC，当然两分钟一次的强制GC依然是存在的

这个GC goal是怎么算出来的，可以看官方的这篇文章 https://tip.golang.org/doc/gc-guide#GOGC 大概是上一轮GC结束时的live heap大小加上上一轮GC到这一轮GC期间使用的heap的大小之和。建议仔细看原文，原文里下面还有个小玩具可以通过控制GOGC的值来直观的看到变化。

因此建议设的内存大小是两倍基础值，例如这里就是37G*2=74G，保证最差情况，在第一次list成功后占用了37G，watch某种原因失败，重新list又需要额外临时占用37G=74G。而不用上述优化list版本，直接使用社区list方案的话，这里则需要基础的37G+临时占用的37*2=111G。

这里最好能使用go1.19引入的[GOMEMLIMIT](https://pkg.go.dev/runtime/debug#SetMemoryLimit)对内存上限进行约束，否则有可能因为goal的不断提升，导致触发oom。还是在 asi_zjk_perf_test01 集群，list需要2分钟才能拿到全量pod，而这个集群每秒的modified事件有4000之多，因此2分钟几乎把大部分pod都更新了一遍，watch此时一定失败，会不停的报错 ` watch of *v1.Pod closed with: too old resource version` 然后一直用得到的resourceVersion重新list，在不设置 `GOMEMLIMIT` 的情况下，内存很快就OOM，如下图所示，第二次relist就挂了

![](/img/2022-10-29-01-18-26.png)

而设置了 `GOMEMLIMIT` 后，很好的控制住了内存消耗

![](/img/2022-10-29-01-20-51.png)

同时不用担心万一`GOMEMLIMIT`设小了怎么办，这里通过将`GOMEMLIMIT=40GiB`进行测试，依然是会使用超过40G的内存，只是GC会变的比较频繁，大概5秒就会进行一次GC。

后续会尝试对informer这个先list后watch的机制进行改进，watch可以用resourceVersion=""的方式先开始，list对数据进行补充即可，不用比较resourceVersion，只要store中已经存在这个数据就可以丢弃掉这条list得到的数据。比较好的是informer本身都是以接口的方式进行暴露，因此有机会在外部重新实现一个informer并以`informerFactory.InformerFor`的方式加入到已有的informerFactory中，对已有的使用方式进行兼容。