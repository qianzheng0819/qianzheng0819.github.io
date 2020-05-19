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
{% highlight c %}
helloService = IHelloService.Stub.asInterface(  
         ServiceManager.getService("hello"));
{% end highlight %}
