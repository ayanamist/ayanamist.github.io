---
layout: post
title: Golang HTTP2实现中内存的不合理使用
---
最近捣腾的项目需要用到HTTP2，就用golang简单撸了一个，做了一些性能测试，GET下性能还好，但POST表单提交的时候，那内存嗖嗖上涨，h2load 10000个连接100个stream per conn下，16G内存没一会就耗干了。

因为刚好提供了HTTP服务，所以直接用[net/http/pprof](https://golang.org/pkg/net/http/pprof/)抓heap出来看，对照着源码，发现居然在有request body的情况下，会无条件的分配64KB的buf，对于一般的表单来说实在是太浪费了。

先简单把这个值缩小到1K，发现内存占用确实有大幅度下降，但request body超过1KB就开始收RST了。再仔细调了调，如果有`Content-Length`且小于64K则直接使用这个，否则就用原来的逻辑。

把这个小改动提了个[PR](https://github.com/golang/go/issues/18509)，不过看golang那边的反应，想做一个更好的实现，只能再等等看了。