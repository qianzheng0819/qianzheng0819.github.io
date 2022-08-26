---
layout: post
title:  "live555之wireshark抓包分析"
date:   2022-08-25 10:39:00 +0800
tags:
  - 音视频开发
description:
---

准备工作
--------------

live555项目的编译是比较简单的，按照官方文档来即可

编译好后cd mediaServer,运行live555MediaServer,记得在目录里放一个小电影。我这里是放得星爷的电影huang.mkv

![p]({{ site.baseurl }}/assets/images/2022-pic/p8.png)

然后打开vlc或者mediaPlayer，看你喜欢哪个。在网络直播流里输入图中的rtsp://...地址，就可以愉快的开始看直播了。

开始抓包
-----------------------
wireshark抓包挺简单的，抓完后输入关键字rtsp，就可以得到想要的包了

![p]({{ site.baseurl }}/assets/images/2022-pic/p9.png)

图中可以看到我们依次收到了客户端的如下请求：

- OPTIONS
- DESCRIPE
- SETUP
- SETUP
- PLAY

后续的文章会结合报文分析RTSPServer如何处理这些请求。
