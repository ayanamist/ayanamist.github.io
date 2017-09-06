---
layout: post
title: Chrome的新建标签页上删除“最常访问”
---

今天更新的Chrome 61，不允许Extension劫持N(ew) T(ab) P(age)的请求了，因此之前所有隐藏最常访问（MostVisited）的Extension都失效了。

但我确实不喜欢显示这个东西，凭什么要把自己的浏览习惯暴露在非常容易被窥视到的地方。

搜来搜去，找到了[这里](https://www.sitepoint.com/community/t/editing-google-chromes-style-file/206881/4)的建议：

用十六进制编辑器修改resources.pak文件，搜索`#most-visited`，把后面的内容替换成`display: none;`，多出来的字符用空格覆盖。这样就默认隐藏了。

还有另一种办法，是直接使用Chromium版本，[这里](https://chromium.woolyss.com/)这里有个列表，其中提供Chromium Stable Portable的是[chrlauncher](https://www.henrypp.org/product/chrlauncher)，也可以尝试
