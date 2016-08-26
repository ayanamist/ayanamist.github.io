---
layout: post
title: 让Metro App走代理
date: '2016-08-06T15:01:34+08:00'
tags:
- proxifier
- Windows
tumblr_url: http://blog.ayanamist.com/post/148532641011
---
用[EnableLoopBack](https://blogs.msdn.microsoft.com/fiddler/2011/12/10/revisiting-fiddler-and-win8-immersive-applications/) 解开不允许使用本地代理的限制。

配置上pac脚本，不要配代理服务器，会导致AnyConnect无法连接VPN。

Proxifier的话，最好把下面几个程序配置进去

<code>svchost.exe;dashost.exe;InstallAgent*.exe;RuntimeBroker.exe;services.exe; unsecapp.exe; WinStore.Mobile.exe; ApplicationFrameHost.exe; ShellExperienceHost.exe; SettingSyncHost.exe; sihost.exe; taskhostw.exe; wininit.exe</code>
