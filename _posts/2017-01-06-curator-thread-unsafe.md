---
layout: post
title: Apache Curator的一处经典的线程不安全问题
---

先看代码截图

![screenshot](/img/2017-01-06_140232.png)

`semaphore.acquire`会在`lease.close`后返回，返回后，同时有两个线程都在对`this.lease`变量进行修改，所以外面的`acquire`方法在返回`true`时，`this.lease`变量有可能是`null`的！！！

于是在接下来的`release`时是无法把之前获取到的`lease`对象进行`lease.close`的，这个锁永远的丢了。

这个问题在最新的curator-recipes上都依旧存在：[InterProcessSemaphoreMutex.java](https://github.com/apache/curator/blob/master/curator-recipes/src/main/java/org/apache/curator/framework/recipes/locks/InterProcessSemaphoreMutex.java)

```
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>3.2.1</version>
</dependency>
```

修改的方法很简单，先上代码：

![screenshot](/img/2017-01-06_150339.png)

注意到，我先把`this.lease`变为局部变量，然后立刻把`this.lease`设为`null`释放掉，接着再去调用`lease.close`，这样`this.lease`这个变量就始终只有一个线程在修改，线程安全了。

问题及相关补丁已经报告给社区：[CURATOR-376](https://issues.apache.org/jira/browse/CURATOR-376)