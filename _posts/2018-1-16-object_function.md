---
layout: post
title:  "翻译一下object类的方法"
date:   2018-1-16 10:10:00 +0800
categories: 学习
tags:   java
description:
---
### void wait(long millis)
当前线程等待直到另外一个线程为该对象执行notify()或者notifyAll()方法，或者一个特定量时间过去。

当前线程必须拥有该对象的监管权利。

该方法导致当前线程把自己放置在该object的等待集中，然后丢弃任何对该对象的同步要求。线程T不再执行计划工作而且保持睡眠直到下列四种情况之一发生：

+ 其他线程执行了该object的notify方法而且线程T碰巧被选为需要唤醒的线程
+ 其他线程对该对象执行了notifyall方法
+ 其他线程打断了线程T
+ 特定时间已经流逝。如果timeout为零，则等待直到唤醒

线程T被唤醒后，从object的等待集合中移除，重新能够做计划工作。它同其他线程一样竞争object的同步权力；一旦线程获取了object的控制权，
