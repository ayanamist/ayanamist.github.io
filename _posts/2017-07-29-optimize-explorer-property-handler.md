---
layout: post
title: explorer.exe打开视频文件夹很慢的问题 
---

如果一个文件夹里有几百个视频文件（avi、wmv、mkv、mp4之类），explorer打开就会很慢，因为有preview，还有property handler（用来支持column能选length、dimension），但如果不需要这些东西，那还是能想点办法提速的。

preview的话在Folder Option里勾上Always show icons, never thumbnails就好了。

property handler比较麻烦，先下载[ShellExView](http://www.nirsoft.net/utils/shexview.html#DownloadLinks)（X64系统请用64位版本），然后找到MF XXX Property Handler那几个，全部disable掉，再在菜单里Options -> Restart Explorer即可。
