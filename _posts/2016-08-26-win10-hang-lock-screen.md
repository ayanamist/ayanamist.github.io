---
layout: post
title: Windows 10 周年更新后，锁屏后一定概率无法登录
date: '2016-08-26T13:18:00+08:00'
tags:
- Windows
---
周年更新后，会有一定概率的出现，锁屏后，再次登录时，转几十秒的圈然后继续提示登录。密码是正确的。此时只能重启。

有网友发现开机之后任务管理器中会出现两个ChsIME.exe进程，前一个属于当前用户，后一个属于system。

目前已经确定，只要结束第二个ChsIME.exe进程（即system用户）就不会出现问题。

如果不想每次开机都要进任务管理器结束该进程，所以一个简单的办法就是：

进入C:\Windows\System32\InputMethod\CHS，找到ChsIME.exe，将system用户的权限设为拒绝即可！

[来源](http://bbs.pcbeta.com/viewthread-1708017-1-1.html)
