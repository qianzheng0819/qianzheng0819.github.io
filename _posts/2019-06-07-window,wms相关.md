---
layout: post
title:  "window,wms相关"
date:   2019-06-07 11:17:00 +0800
categories: android
tags:   android
description:
---

通过mWindowSession来完成Window的添加过程 ，mWindowSession的类型是IWindowSession，是一个Bindler对象，真正的实现类是Session，也就是Window的添加是一次IPC调用。(mWindowSession在ViewRootImpl的构造函数中通过WindowManagerGlobal.getWindowSession();创建)

同时将mWindow（即 W extends IWindow.Stub）发送给WmS，用来接收WmS信息。

![p1]({{ site.baseurl }}/assets/images/2019-06-07-pic/p1.png)
