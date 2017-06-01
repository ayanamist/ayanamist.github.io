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
