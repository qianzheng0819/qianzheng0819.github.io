---
layout: post
title:  "HttpUrlConnection源码浅谈"
date:   2021-01-15 19:37:00 +0800
categories: android
tags:   android
description:
---

### **前言**
熟悉android源码的同学应该知道HttpUrlConnection的底层实现从android4.4开始就由httpclient   
替换成了okhttp。所以在分析HttpUrlConnection源码的同时，也是在阅读okhttp的源码。分析的方法   
是先画出时序图，然后结合时序图去分析关键方法中的关键代码。   

{%highlight java%}
URL url = new URL("https://certs.cac.washington.edu/CAtest/");
HttpsURLConnection urlConnection = (HttpsURLConnection)url.openConnection();
//connect()方法不必显式调用，当调用conn.getInputStream()方法时内部也会自动调用connect方法
urlConnection.connect();
//调用getInputStream方法后，服务端才会收到完整的请求，并阻塞式地接收服务端返回的数据
InputStream in = urlConnection.getInputStream();
{%endhighlight%}
本文基于android4.4.4源码分析，各个版本代码略有不同。  

---

### **时序图**

### **关键方法**
**1.new URL(String spec)**
![p1]({{ site.baseurl }}/assets/images/2021-pic/p1.jpg)

**2.setupStreamHandler()**
![p]({{ site.baseurl }}/assets/images/2021-pic/p2.png)
可以看到选择的是HttpHandler

**13.initHttpEngine()**
![p]({{ site.baseurl }}/assets/images/2021-pic/p3.png)
