---
layout: post
title:  "Android应用程序消息处理机制（Looper、Handler）分析"
date:   2020-05-20 14:57:00 +0800
categories: android
tags:   android
description:
---

### MessageQueue数据结构
是一个以msg.when升序排序的单链表队列，mMessage记录队列头；

### epoll过程
有一个管道，当有msg时会向写端口写数据，epoll监听读事件返回。

在该消息处理机制中，epoll主要起到为messagequeue实现阻塞功能。
Looper.loop开启true循环，调用queue.next。第一次调用是epoll_wait(0),立即返回队列头mMessage。
如果队列为空，调用epoll_wait(-1)即为无限阻塞直到有新的消息到达；如果队列非空，但是取得的队列头mMessage.when高于now,则
调用epoll_wait(when - now)，这时它又起到一个定时作用。而且巧妙的是，定时等待中还能接受
新的消息。如果mMessage.when低于now，就返回目标msg回到上一层循环继续queue.next取消息。注意
是有两层循环的，第二层循环返回msg才会停止回到上层循环，代码中很清晰。

Android应用程序消息处理机制还是挺巧妙的。
