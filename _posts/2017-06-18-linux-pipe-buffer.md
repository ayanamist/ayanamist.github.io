---
layout: post
title: linux上pipe的一些处理 
---

帮人写个命令通道，反馈说`sh -c "tail -F /tmp/log | awk '{print \$0}'"`这样的命令，在终端下能输出数据，但用我的程序却什么都不输出。

于是调试了下，发现`sh -c "tail -F /tmp/log | awk '{print \$0}'" | cat`后面加个cat命令后终端下也无法输出了。

在两个群里问了下，总算搞清楚了，linux下pipe有buffer，但buffer的行为可以控制，终端下stdout是pty，默认是line buffered，所以没问题，输出到pipe，pipe是full buffered，刚好数据量又不大，就全憋buffer里导致不输出了。

给了几个方案，一个是使用`awk '{print \$0; fflush(stdout)}'`和`grep --line-buffered`这样的方案，但对执行的命令有侵入性；还有一个方案就是使用[stdbuf](https://linux.die.net/man/1/stdbuf)命令`stdbuf -oL -eL sh -c`这样执行。

搜了下stdbuf的原理，刚好有[一篇文章](https://tomli.blog/archives/2014/04/1832.html)介绍，大概就是这个命令修改了`LD_PRELOAD`注入了libstdbuf，并且通过`_STDBUF_O`等环境变量控制libstdbuf的行为；libstdbuf中利用[setvbuf](https://linux.die.net/man/3/setvbuf)在程序启动前把stdout的buffer模式修改掉，从而替程序做了修改。

考虑到libstdbuf是coreutils的一部分，一般都自带了，所以我也实现了同样的逻辑，判断了libstdbuf.so文件存在的话，则执行相同的环境变量修改，这样就不用依赖stdbuf程序来注入libstdbuf了。
