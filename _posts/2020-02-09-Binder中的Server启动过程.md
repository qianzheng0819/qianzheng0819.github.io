---
layout: post
title:  "Binder中的Server启动过程"
date:   2020-02-09 11:17:00 +0800
categories: android
tags:   android
description:
---

记录一下源码中的重点。

在MediaPlayerService向ServiceManager注册服务时，写入Parcel内容：
writeInt32(IPCThreadState::self()->getStrictModePolicy() | STRICT_MODE_PENALTY_GATHER);
writeString16("android.os.IServiceManager");
writeString16("media.player");
writeStrongBinder(new MediaPlayerService())

IPCThreadState::transact的核心代码为IPCThreadState::waitForResponse,以及waitForResponse
下的talkWithDriver()方法。

IPCThreadState::transact先写数据writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL)，
其中data就对应上面的Parcel。然后进入waitForResponse，进入talkWithDriver。

talkWithDriver调用ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)，bwr是读写数据的集合体。
先写数据，依次调用内核binder.c->ioctl->binder_thread_write,新建binder_transaction_data结构体tr数据，
tr直接复制了bwr里mOut内容即写数据。然后在内核空间分配内存binder_transaction t,t记录了target_proc,target_thread
等信息，t->buffer复制了tr数据。tr是用户空间内存，而t是内核空间内存。

关键信息：
target_node = binder_context_mgr_node;
target_proc = target_node->proc;
target_list = &target_proc->todo;
target_wait = &target_proc->wait;
thread->transaction_stack = t;该thread是ServiceManager进程binder_get_thread()分配。
附：ProcessState作为全局进程上下文，是单例模式叫做gProcess.它是ServiceManager在注册成为binder大管家时，open_driver()
生成filep.而filep->proc即为ServerManager进程。所以ServiceManager既是服务，又是binder的守护进程，非常重要。
下文中大部分proc都是ServerManager进程。

binder_thread_write调用的binder_transaction()在最后调用了：
  t->work.type = BINDER_WORK_TRANSACTION;
  list_add_tail(&t->work.entry, target_list);
  tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
	list_add_tail(&tcomplete->entry, &thread->todo);
	if (target_wait)
		wake_up_interruptible(target_wait);

其中target_wait=traget_proc_wait,即为唤醒了ServiceManager在binder_loop()->binder_read()
时的睡眠。ServiceManager得到上面的t，即得到了数据。

插入以下，中间binder_thread_read()会分别读到两个int32的cmd,BR_LOOP和BR_TRANSACTION_COMPLETE

![p2]({{ site.baseurl }}/assets/images/2019-pic/p2.png)
