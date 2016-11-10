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

Update:

上面其实不是最好的办法，最标准的姿势，应该是`go build`中指定`-tags 'netcgo'`，按照[src/net/conf_netcgo.go](https://golang.org/src/net/conf_netcgo.go)中的指定，就会在编译期就使用cgo resolver。

但这会带来另外一个问题，公司里用的是魔改RHEL5，内核上到2.6.32了，但默认的gcc还停留在4.1，按照[golang/go#9520](https://github.com/golang/go/issues/9520)的描述，会导致编译错误，解决方法是修改golang的源代码，将[src/cmd/go/build.go](https://golang.org/src/cmd/go/build.go#L3630)的`disableBuildID`方法内容，按上面的注释，从`"-Wl,--build-id=none"`修改为`"-r"`，然后重新编译出`go`可执行文件并替换，注意根据[文档](https://golang.org/doc/install/source#environment)指示设定`$GOROOT_FINAL`变量。
