---
layout: post
title: java.nio.Charset一处反直觉设计 
---

```java
new String("foobar".getBytes("UTF-8"), "UTF-8")
```

这里面是两个非常常见的String和byte数组之间的转换，还有另一种写法

```java
Charset charset = Charset.forName("UTF-8");
new String("foobar".getBytes(charset), charset);
```

直觉上第二种写法会好一点，因为直接上会觉得第一种写法的背后也是去调`Charset.forName`方法。

但看了内网的帖子后才发现，居然第一种写法更好，Charset去做encode和decode时会生成一个CharsetEncoder和CharsetDecoder对象，在第一种写法里，这个对象会和和其它信息一起组成一个java.lang.StringCoding.StringEncoder类，然后用ThreadLocal<SoftReference<StringEncoder>>缓存起来，这样在同一个线程里就不会反复生成，而第二种算法却是每调一次就生成一个对象，性能反而低下。

这个设计比较反直觉，在此记一笔。
