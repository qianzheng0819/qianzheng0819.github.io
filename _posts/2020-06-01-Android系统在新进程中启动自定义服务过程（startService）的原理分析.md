---
layout: post
title:  "Android系统在新进程中启动自定义服务过程（startService）的原理分析"
date:   2020-06-01 16:34:00 +0800
categories: android
tags:   android
description:
---

详细代码分析可以参考罗神的博客，我这里总结一下方便记忆。

主进程和ams ipc创建新进程pid和新服务serviceRecord,并分别保存它们在ams服务的集合里。

新进程会加载ActivityThread，然后流程如下：
{%highlight java%}
public final class ActivityThread {

	......

	public static final void main(String[] args) {

		......

		Looper.prepareMainLooper();

		......

		ActivityThread thread = new ActivityThread();
		thread.attach(false);

		......

		Looper.loop();

		......

		thread.detach();

		......
	}

  public final class ActivityThread {

	......

	private final void attach(boolean system) {

		......

		if (!system) {

			......

			IActivityManager mgr = ActivityManagerNative.getDefault();
			try {
				mgr.attachApplication(mAppThread);
			} catch (RemoteException ex) {
			}
		} else {

			......

		}

		......

	}

	......

}
}
{%endhighlight%}

新进程会和ams ipc通过pid,processName取回对应的服务。然后ams会调用realStartServiceLocked，
ipc ActivityThread的ApplicationThread服务（ApplicationThread是framework java层的服务，不要被它的名字迷惑^-^）。上述过程有两次ipc。

我的感觉：ams好像一个临时中转站，在这里生产出目标ServiceRecord，然后将它入库封存。新进程要取用ServiceRecord时，通过pid找到进程app，再用app.processName和app.uid找到ServiceRecord。
期间有主进程向ams的ipc，新进程向ams的ipc，ams向新进程的ipc三次ipc。

最后贴下罗神的总结：
一. Step 1至Step 7，从主进程调用到ActivityManagerService进程中，完成新进程的创建；

二. Step 8至Step 11，从新进程调用到ActivityManagerService进程中，获取要在新进程启动的服务的相关信息；

三. Step 12至Step 20，从ActivityManagerService进程又回到新进程中，最终将服务启动起来
