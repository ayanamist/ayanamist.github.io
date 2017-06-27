---
layout: post
title: 匪夷所思的golang程序panic 
---

我的程序布在线上几十万机器上，考虑到环境的差异，每个goroutine启动前都会加段保护，通过`recover`捕捉可能的panic然后告警出来。

之前就在线上就抓到一个非常匪夷所思的panic在启动进程的地方，[报给golang官方](https://github.com/golang/go/issues/19918)也没个所以然，怀疑是内存损坏。

今天又遇到了一例同样匪夷所思的，趁机记录下来，同样怀疑是内存损坏，因为[类似的代码](https://play.golang.org/p/gvq0_7vXTv)按golang语法定义是不可能造成panic的

告警的栈是：

![screenshot](/img/2017-06-01_201705.jpg)

对应的golang代码是：

![screenshot](/img/2017-06-01_201630.jpg)

Update on 2017-06-15 继续记录奇奇怪怪的panic

panic的栈如下图

![screenshot](/img/2017-06-15_094934.png)

对应go代码如下图，除了内存坏了，我是找不到其它导致`panic runtime error: integer divide by zero`的原因

![screenshot](/img/2017-06-15_094958.png)

Update on 2017-06-27

panic的栈如下图，挂在strconv.ParseInt里

![screenshot](/img/2017-06-27_183111.png)

对应代码位置，看起来是str部分被破坏了，当然没用unsafe的东西乱改啦

![screenshot](/img/2017-06-27_183224.png)
