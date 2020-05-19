---
layout: post
title:  "Binder中的Server启动过程"
date:   2020-05-18 11:17:00 +0800
categories: android
tags:   android
description:
---

一.获取Service Manager的Java远程接口
关键代码是ServiceManagerNative.asInterface(BinderInternal.getContextObject())
中间源码分析比较简单省略，直接记录关键点。BinderInternal.getContextObject()相当于
new BinderProxy(),该proxy的field mObject记录了c++层的BpBinder(0);c++层的BpBinder,
BBinder,binder驱动之间的关系以前已经探讨过，这里不再写。
