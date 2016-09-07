---
layout: post
title: golang使用cgo dns resolver
tags:
- golang
---
golang在Linux下在nsswitch.conf允许的情况下，默认会使用自己的pure go resolver，parse了`/etc/resolv.conf`的内容。

但在部分安装了nscd的机器上，如果刚好`resolv.conf`中配置了无效的dns服务器，则ping和curl命令都可以通过nscd的缓存获取到解析结果，而golang的程序则会报错解析失败。

按照golang的[官方文档](https://golang.org/pkg/net/#hdr-Name_Resolution)，可以通过设置`GODEBUG=netdns=cgo`来强制使用libc的resolver，go代码中则可以通过以下代码开启

{% highlight go %}
import "os"

func init() {
	os.Setenv("GODEBUG", "netdns=cgo")
}
{% endhighlight %}
