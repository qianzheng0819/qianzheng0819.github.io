---
layout: post
title:  "Binder机制在应用程序框架层的Java接口分析"
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

二.HelloService的启动过程
该HelloService的demo在老罗博客里有，该博客也主要是记录老罗博客的关键点，方便我自己
记忆。HelloService在SystemServer.java里启动ServerThread启动起来，其他的常见framework
层java服务如Ams,Wms也是在这里启动起来。
new HelloService();这个语句会调用HelloService类的构造函数，而HelloService类继承于IHelloService.Stub类，IHelloService.Stub类又继承了Binder类，因此，最后会调用Binder类的构造函数：
{% highlight java %}
public class Binder implements IBinder {
......

private int mObject;

......


public Binder() {
  init();
  ......
}


private native final void init();


......
}
{% endhighlight %}
这里调用了一个JNI方法init来初始化这个Binder对象：

{% highlight c %}

static void android_os_Binder_init(JNIEnv* env, jobject clazz)
{
    JavaBBinderHolder* jbh = new JavaBBinderHolder(env, clazz);
    if (jbh == NULL) {
        jniThrowException(env, "java/lang/OutOfMemoryError", NULL);
        return;
    }
    LOGV("Java Binder %p: acquiring first ref on holder %p", clazz, jbh);
    jbh->incStrong(clazz);
    env->SetIntField(clazz, gBinderOffsets.mObject, (int)jbh);
}
{% endhighlight %}
JavaBBinderHolder这个对象就是在java和c++中切换的一个Holder，holder构造函数传入了clazz,其中clazz就是我们的关键service。后面通过ibinderForJavaObject操作holder把java端服务binder转换为c++端JavaBBinder

 三. Client获取HelloService的Java远程接口的过程
{% highlight java %}
helloService = IHelloService.Stub.asInterface(  
         ServiceManager.getService("hello"));
{% endhighlight %}
相当于
 {%highlight java%}
 helloService = IHelloService.Stub.asInterface(new BinderProxy()));
 // BinderProxy的mObject记录了BpBinder(handler),handler就是对应服务的底层句柄
 {%endhighlight%}

 BinderProxy.transact函数是一个JNI方法，我们在前面已经介绍过了，这里不再累述。最过调用到Binder驱动程序，Binder驱动程序唤醒HelloService这个Server。前面我们在介绍HelloService的启动过程时，曾经提到，HelloService这个Server线程被唤醒之后，就会调用JavaBBinder类的onTransact函数：
 {%highlight c++%}
 class JavaBBinder : public BBinder
 {
 	JavaBBinder(JNIEnv* env, jobject object)
 		: mVM(jnienv_to_javavm(env)), mObject(env->NewGlobalRef(object))
 	{
 		......
 	}

 	......

 	virtual status_t onTransact(
 		uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags = 0)
 	{
 		JNIEnv* env = javavm_to_jnienv(mVM);

 		......

 		jboolean res = env->CallBooleanMethod(mObject, gBinderOffsets.mExecTransact,
 			code, (int32_t)&data, (int32_t)reply, flags);

 		......

 		return res != JNI_FALSE ? NO_ERROR : UNKNOWN_TRANSACTION;
 	}

 	......

         JavaVM* const   mVM;
 	jobject const   mObject;
 };
 {%endhighlight%}
 前面我们在介绍HelloService的启动过程时，曾经介绍过，JavaBBinder类里面的成员变量mObject就是HelloService类的一个实例对象了。因此，这里通过语句：
 {%highlight c++%}
 jboolean res = env->CallBooleanMethod(mObject, gBinderOffsets.mExecTransact,
			code, (int32_t)&data, (int32_t)reply, flags);
 {%endhighlight%}
 就调用了HelloService.execTransact函数，而HelloService.execTransact函数继承了Binder类的execTransact函数：
{%highlight c++%}
public class Binder implements IBinder {
	......

	// Entry point from android_util_Binder.cpp's onTransact
	private boolean execTransact(int code, int dataObj, int replyObj, int flags) {
		Parcel data = Parcel.obtain(dataObj);
		Parcel reply = Parcel.obtain(replyObj);
		// theoretically, we should call transact, which will call onTransact,
		// but all that does is rewind it, and we just got these from an IPC,
		// so we'll just call it directly.
		boolean res;
		try {
			res = onTransact(code, data, reply, flags);
		} catch (RemoteException e) {
			reply.writeException(e);
			res = true;
		} catch (RuntimeException e) {
			reply.writeException(e);
			res = true;
		} catch (OutOfMemoryError e) {
			RuntimeException re = new RuntimeException("Out of memory", e);
			reply.writeException(re);
			res = true;
		}
		reply.recycle();
		data.recycle();
		return res;
	}
}
{%endhighlight%}
这里又调用了onTransact函数来作进一步处理。由于HelloService类继承了IHelloService.Stub类，而IHelloService.Stub类实现了onTransact函数，HelloService类没有实现，因此，最终调用了IHelloService.Stub.onTransact函数：
{%highlight c++%}
public interface IHelloService extends android.os.IInterface
{
	public static abstract class Stub extends android.os.Binder implements android.os.IHelloService
	{
		public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
		{
			switch (code)
			{
			......
			case TRANSACTION_getVal:
				{
					data.enforceInterface(DESCRIPTOR);
					int result = this.getVal();
					reply.writeNoException();
					reply.writeInt(result);
					return true;
				}
			}
			return super.onTransact(code, data, reply, flags);
		}

		......

	}
}
{%endhighlight%}
