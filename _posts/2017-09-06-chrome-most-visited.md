---
layout: post
title: Chrome的新建标签页上删除“最常访问”
---

今天更新的Chrome 61，不允许Extension劫持N(ew) T(ab) P(age)的请求了，因此之前所有隐藏最常访问（MostVisited）的Extension都失效了。

但我确实不喜欢显示这个东西，凭什么要把自己的浏览习惯暴露在非常容易被窥视到的地方。

搜来搜去，找到了[这里](https://www.sitepoint.com/community/t/editing-google-chromes-style-file/206881/4)的建议：

用十六进制编辑器修改resources.pak文件，搜索`#most-visited`，把后面的内容替换成`display: none;`，多出来的字符用空格覆盖。这样就默认隐藏了。

多尝试了一下，可以通过修改resources.pak两处，完整的去掉NTP内容但保留书签栏：

1. 搜索`chrome-search://local-ntp/local-ntp.js`，将整个字符串替换为空格。这样可以禁止Chrome从Google服务器上加载NTP。
2. 搜索`<link rel="stylesheet" href="chrome-search://local-ntp/local-ntp.css"></link>`修改为`<style>body{display:none;}</style>`，不足的地方用空格补足，这样完全禁止本地NTP的显示。
