---
layout: post
title: cygwin编译nghttpx
date: '2015-12-23T13:26:30+08:00'
tags:
- nghttpx
- cygwin
- spdylay
tumblr_url: http://blog.ayanamist.com/post/135758773486
---
起因是想用h2代理了，但golang上h2支持还很差，两个spdy库折腾了一晚上都没办法很好的支持CONNECT语义，只好退而求其次用nghttp方案。

2017-11-06更新：为了编译出一个静态链接的版本，需要自己编译依赖，因为cygwin中带的库都没有静态链接的版本。
- zlib：需要注意使用[这个patch](https://github.com/madler/zlib/issues/268#issuecomment-336765596)

先按[这里](http://stackoverflow.com/a/27389613/522024)把FD_SETSIZE调大，否则会经常出现莫名其妙的连接不上的问题。

编译主要[这里](https://nghttp2.org/documentation/package_README.html#building-from-git)，注意文档中提示的[几个点](https://nghttp2.org/documentation/package_README.html#notes-for-building-on-windows-mingw-cygwin)。

<del>而编译nghttp时则有[两个小坑](https://github.com/tatsuhiro-t/nghttp2/issues/108#issuecomment-166802587)</del>其中一个问题已修正，同时提交的[一个PR](https://github.com/tatsuhiro-t/nghttp2/pull/461)也合并了；另一个则是需要注意设置</p>

`export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig`

防止找不到依赖。

编译后配置文件可以为

```
frontend=127.0.0.1,8080
backend=xxxx.com,443
client-proxy=yes
add-x-forwarded-for=no
no-via=yes
no-ocsp=yes
npn-list=h2
insecure=yes
workers=4
```

注意文件要保存成Unix换行符，否则在shrpx里会提示Port is invalid
