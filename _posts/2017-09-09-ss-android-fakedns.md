---
layout: post
title: ss-android上实现fakedns 
---

ss-android算是android上比较稳定的产品了，但对dns的处理上依然不够理想，经常因为dns解析阻塞导致请求缓慢甚至失败。

ss-android对dns解析，有两种方案：
1. udp方案，直接走ss的udp转发，需要服务端也支持udp转发，而且从我个人使用来说，并不十分稳定。
2. tcp方案，用overture将请求转成tcp，再从tcp连接出去，缺点非常明显，原本udp的一来一回，到这里需要三步握手四步挥手。

iOS上surge的[fakedns的方案](https://trello.com/c/AFX8ht38)非常好，毕竟dns污染的域名是少数的，大部分人在手机上的操作也不是那么复杂，完全可以通过定制规则的方式来解决这个问题。

接下来就是怎么实现了，修改[ss-libev](https://github.com/shadowsocks/shadowsocks-libev)的方案我觉得略微复杂了一点，因此决定修改[go-ss2](https://github.com/shadowsocks/go-shadowsocks2)，直接添加相关代码。

通读了一下ss-android和ss-libev的代码，大概注意的点如下：
1. ss-libev通过[android.c](https://github.com/shadowsocks/shadowsocks-libev/blob/master/src/android.c)文件提供android定制支持
2. 通过`__ANDROID__`宏、-V参数来控制部分功能，主要是禁用socks5 domain支持、回调java的protect和stat
3. android的VPNService需要用[protect](https://developer.android.com/reference/android/net/VpnService.html#protect(int))方法对vpn app自己的连接进行保护，防止陷入死循环，相关原理见这个[SO答案](https://stackoverflow.com/a/38764232/522024)。ss-libev是通过一个unix socket，将fd用libancillary传给java端，[java端监听请求并将fd提取出来，完成protect后再返回结果](https://github.com/shadowsocks/shadowsocks-android/blob/master/mobile/src/main/scala/com/github/shadowsocks/ShadowsocksVpnThread.scala)。
4. android的速度和流量信息，也是通过另一个unix socket文件，从libev回调回来的。

于是就照着在go-ss2上实现类似的东西就可以了，修改后的代码在[这里](https://github.com/ayanamist/go-shadowsocks2/tree/ss-android)，过程中几点要注意的：
1. 需要安装[android-ndk](https://developer.android.com/ndk/downloads/index.html#stable-downloads)来编译android版本
2. 生成各个平台的toolchain（以下均以arm64说明）：`$ANDROID_NDK_HOME/build/tools/make_standalone_toolchain.py --arch arm64 --api 21 --install-dir $ANDROID_ARM64_TOOLCHAIN`
3. 导出环境变量
    ```bash
    CGO_ENABLED=1
    CC=$ANDROID_ARM64_TOOLCHAIN/bin/aarch64-linux-android-clang
    GOOS=android
    GOARCH=arm64
    ```
3. 最好使用golang 1.9进行编译，本文写于该版本
    由于golang的限制，按[这里的方法](https://github.com/golang/go/issues/21820#issuecomment-328281230)新建文件，然后重新编译标准库`go install -v net`
4. `go install -v github.com/ayanamist/go-shadowsocks2`，将文件改名为libss-local.so，替换到apk包的lib/arm64-v8a目录下
5. 编写一个空壳文件liboverture.so，也替换到apk中，屏蔽掉原本的overture，dns功能由新的go-ss2程序提供
    ```
    #!/system/bin/sh
    exec sleep 10000000
    ```
6. 用dex2jar的d2j-apk-sign签名后安装
7. 调试的时候一定要关掉selinux，否则各种奇奇怪怪的问题：`setenforce permissive`；为了不修改java代码，因此fakedns使用了[11.0.0.0/8](https://en.wikipedia.org/wiki/List_of_assigned_/8_IPv4_address_blocks#List_of_assigned_.2F8_blocks_to_the_United_States_Department_of_Defense)的地址段。

使用的注意事项：
1. 需要fakedns的，用ss-android的自定义规则，将域名添加进去，符合规则的都会走fakedns，无法关闭
2. 需要禁用DNS转发，go-ss2不支持udp-relay，所以开启DNS转发的话会无法解析
3. 建议将路由设置为“绕过局域网及中国”获得更好体验
4. 不支持IPv6
5. go-ss2内置成114DNS，android上设置的远端DNS无效。事实上原始版本，如果不开DNS转发，也是走114和dnspod
    可以通过访问[welcome.114dns.com](http://welcome.114dns.com/)来检查是否生效
    可以访问[www.taobao.com](https://www.taobao.com/)看是否会被重定向到国际版，原来的DNS转发由于服务器一般在国外，会解析出国外的IP导致访问国际版页面。
6. 由于本人使用MIUI，因此不需要流量速度显示，因此没有实现这部分。
7. 目前发现，如果在开启ss-android的情况下，在另一个vpn app中抢夺vpn，那后续ss-android会出现开启后无法访问网络的问题，请执行一次“强制停止”后重新开启。

最后附一个ss-android_v4.1.8的fakedns版，注意只替换了arm64：[apk下载](https://github.com/ayanamist/ayanamist.github.io/releases/download/ss-android-fakedns_v4.1.8/Shadowsocks_v4.1.8_fakedns.apk)
