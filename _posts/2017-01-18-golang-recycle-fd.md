---
layout: post
title: Golang小坑：gc时会回收未引用且未关闭的fd
---

如标题所述，也是今晚调试一晚上的发现，当一个fd被打开后，没有任何对象引用了它，在gc时，fd会被关闭。

踩坑的原因是，常用flock一个pid文件来做进程互斥，但跑着跑着，互斥就没了，查了半天才发现被flock的fd消失了，自然互斥就没有了。

见如下代码，执行后，用`ls -l /proc/{pid}/fd`去查看fd打开情况，会发现open的那个fd不见了；而把`runtime.GC()`这行注释掉后，fd就会依旧存在。

{% highlight go %}
package main

import (
	"os"
	"log"
	"time"
	"runtime"
)

func openFile(path string) error {
	_, err := os.Open(path)
	return err
}

func main() {
	if err := openFile(os.Args[1]); err != nil {
		log.Fatal(err)
	}
	// trigger GC below will also recycle the non-referenced fd opened before
	runtime.GC()
	time.Sleep(time.Hour)
}
{% endhighlight %}

这个行为应该是非常罕见的设计，因为其它带gc的语言，这种情况往往会出现fd泄漏。在Google搜了下，也有人[遇到了类似的网络连接被关闭](http://blog.csdn.net/wang_xijue/article/details/52013262)的情况。

原来golang也有个类似java的finalizer的设计，可以通过`runtime.SetFinalizer`进行注册，在golang标准库里，打开的文件、网络连接、进程都会被注册上finalizer在gc时进行释放。

这个设计比较考虑低级错误，但对有意的fd释放，就只好显式的加一个变量保持引用了。
