---
layout: post
title: 修改Inoreader for Android在列表模式下title的最大行数
---

Inoreader各种好，以前用iOS版的时候，有个选项可以显示完整title，但Android版却不行，最大显示3行，超出就截断，我给他们写邮件要求和iOS版平权，人家表示这个有排期但没ETA，这就等于变相的说我们不会做。

一般来说3行确实够用了，但我把微博、Twitter的原始内容也作为title了，这样就不够了。既然官方不肯做，那就自己动手丰衣足食。

照例，用[apktool](https://ibotpeaches.github.io/Apktool/)解包，然后到res目录下找到item_article_list.xml文件里的`:maxLines="3"`这一行，改成300，然后保存，在打包回去，再用[dex2jar](https://github.com/pxb1988/dex2jar/releases)的2.1-nightly-26版中的d2j-apk-sign签名工具签个名就完事。

如果对apktool打的包不放心，可以把打包后apk文件里res目录下，修改了的那几个文件替换进原始apk中，再签名。
