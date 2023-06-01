---
layout: post
title:  "浅谈Service Manager成为Android进程间通信（IPC）机制Binder守护进程之路"
date:   2023-06-1 10:11:00 +0800
categories: android
tags:   Framework
description:
---

前言
----------
老罗的android之旅系列博客，应该是不少人入门framework之选

这系列的博客，除去sufaceFlinger和wms相关，我基本上都已经阅读过2-3次

当然也收获良多，理解了binder机制，理解了activity的启动流程，理解了activity的实现框架等等

在阅读binder源码时，往往有那种恍然大悟的愉悦感

但是阅读过后一个月，记忆里就所剩无多，只剩下类似BpBinder,BnBinder,ProcessState,

IPCThreadState这些重要类的印象。好记忆不如烂笔头，这就是我写下这系列博客的原因。
