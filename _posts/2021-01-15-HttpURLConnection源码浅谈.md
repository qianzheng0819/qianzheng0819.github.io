---
layout: post
title:  "HttpUrlConnection源码浅谈"
date:   2021-01-15 19:37:00 +0800
categories: android
tags:   android
description:
---

#### 前言
熟悉android源码的同学应该知道HttpUrlConnection的底层实现从android4.4开始就由httpclient
替换成了okhttp。所以在分析HttpUrlConnection源码的同时，也是在阅读okhttp的源码。分析的方法
是先画出时序图，然后结合时序图去分析关键方法中的关键代码。

#### 时序图

#### 关键方法
