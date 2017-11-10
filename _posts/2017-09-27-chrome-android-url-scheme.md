---
layout: post
title: Chrome Android上智能跳转app
---

需求蛮简单，在app存在的时候自动跳转到app里，不存在时跳转到网页版。

之前大家都是用iframe来玩的，但Chrome 25开始就[禁掉了这种方法](https://developer.chrome.com/multidevice/android/intents)。但我实测，在Chrome 61.0.3163.98上修改`location.href`还能跳转。

先放详细的改动，见[这个commit的这个文件](https://github.com/ayanamist/SinaWeiboRSS/commit/c00c9ae93ffe74d440d8d7744e37c0cecf0adde9#diff-bec7118b76cbdeff33f8487ed9580125)。

传统的用`setInterval`加`setTimeout`的方式不灵了，至少在头几秒里setInterval还是会在切后台后正常运行。而html5 visibility相关api即使弹出app后也依然会显示visible，测了半天，似乎只有注册blur事件，才能感知到离开页面。

我自己的需求比较简单，复杂页面的话，可以在跳转前赋值个flag，然后blur检测这个flag防止误伤。
