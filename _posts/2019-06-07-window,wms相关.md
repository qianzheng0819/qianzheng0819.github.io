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

### InputManager和系统输入相关
[epoll机制](https://blog.51cto.com/yaocoder/888374)
epoll检测缓存区非满和缓存区非空事件,得到有效流,使用非阻塞忙轮询方法来处理这些有效流.对比非阻塞忙轮询方法,阻塞式io缺点是一个线程只能处理一个流,而fork和pthread_create的效率不高.
epoll前身有select,poll方法,其本质一致,检测到流中有io事件便会触发.epoll方法是对poll方法的优化.

[android启动流程](https://juejin.im/post/5b7e72bbe51d453894001ef0)
系统从kernal到native到javaFramework依次启动,Zygote是第一个Java进程,SystemServer是Zygote fork而来的第二个Java进程.由SystemServer进程来启动我们常用的各种服务如ams,pms,wms,ims等
