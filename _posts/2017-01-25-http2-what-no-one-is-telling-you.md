---
layout: post
title: 关于HTTP2，你所不了解的东西
---

这是一篇笔记，原始视频为：[DrupalCon Dublin 2016 HTTP 2 what no one is telling you](https://www.youtube.com/watch?v=QrnpkOHyciU)

略过前面对HTTP2的一些介绍和优势（这个已经烂大街了），后面对各种场景进行H1 vs H2，结论：
1. 丢包（Packet Loss Rate，PLR）对H2的影响要大得多，超过2%的丢包率已经可以让H2比H1显著性的慢。
2. 延迟（Latency）的升高对H2的影响也要大得多，超过300ms后已经非常显著。
3. 建议通过CDN使用H2，由于CDN就近提供服务的特点，可以把上面两个问题的影响降到最低，充分发挥H2的优势。
4. 简单谈了下QUIC，但目前除了Google没有其它有应用。

就我个人对H2使用的经验，分析如下：
1. 丢包的影响主要问题出在H2单tcp连接的设计上，一旦丢包，这个连接上所有stream都要阻塞，而H1多个连接互相隔离，受到的影响就没有那么大。
2. 延迟的影响主要问题出在7层和3层都有ack的设计上，3层tcp收发包都要做ack操作，而H2在7层协议上的Window Update事实上也充当了ack的作用，而连接刚建立时需要互相互相发SETTINGS，在RTT比较高的时候相对于H1也大幅度延长了发第一个请求之前的时间。

接下来谈了Server Push，提到Push机制对现有的浏览器端缓存的整体设计提出了巨大挑战，很容易重复push已经有缓存的数据导致事实上的时间增加和流量浪费，目前主要有两种解决方案：h2o的[casper方案](https://h2o.examp1e.net/configure/http2_directives.html#http2-casper)，以及[Cache-Digests](https://tools.ietf.org/html/draft-kazuho-h2-cache-digest-00)及[变种](https://www.shimmercat.com/en/docs/1.5/caching/#caching-and-http-2-push)，总的思路都是用cookie标记之前的缓存状态；另外我个人也觉得Push机制对服务端提出了很高的要求，因为要push什么都要服务端事先算好，这对于一个大型网站模块化非常厉害的情况下，几乎是办不到的。

接下来谈到HPACK压缩：
1. header压缩看起来很美好，但几乎所有浏览器都用了RFC提议的4KB作为table大小，而演讲者举了个例子，Twitter的CSP这一个header就有2.2KB大，而且这个压缩只对incoming request有意义，对outgoing response就看不出显著的区别来（举了[dropbox的例子](https://blogs.dropbox.com/tech/2016/05/enabling-http2-for-dropbox-web-services-experiences-and-observations/)）。
2. 这个压缩也是per connection的，因此每一个新连接都要从头开始。
3. RFC的描述比较含糊导致实现上可以像[这篇论文](https://www.imperva.com/docs/Imperva_HII_HTTP2.pdf)里提到的创建一个超大header然后疯狂引用这个header导致服务端在应对极少的请求时因为解压缩而耗费巨大内存；事实上这篇论文里还有一些其它有意思的针对H2各个实现的攻击，我前面的[文章](/2017/01/07/golang-h2-memory.html)也提到golang的实现导致内存消耗会非常的大，实现上的问题只能靠时间来解决了。
4. 最后一点提到无法关闭hpack，会导致pipelining无法使用，这个我没太明白，speaker没细说，我也没搜到这方面的文章。
