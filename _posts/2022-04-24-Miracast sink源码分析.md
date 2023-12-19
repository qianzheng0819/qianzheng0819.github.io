---
layout: post
title:  "Miracast sink源码解析"
date:   2022-03-25 11:39:00 +0800
categories: Miracast
tags: Miracast
description:
---

### 前言  
这不是一个教学贴，更多是一个研究源码的笔记博客。      
这段时间主要研究基于miracast的wifi-display实现。miracast分source端和sink端，现在的手机一般都内置了source端。    

我们先研究sink端源码。     

miracast过程中一直会有3个socket:   
1. socket for rtsp   
2. socket for rtp  
3. socket for rtcp    

下面是三个协议的wiki介绍，能让我们了解三种协议的责任，和所要解决的问题。   

**Rtsp**:   
实时流协议（Real Time Streaming Protocol，RTSP）是一种网络应用协议，专为娱乐和通信系统的使用，以控制流媒体服务器。该协议用于创建和控制终端之间的媒体会话。媒体服务器的客户端发布VCR命令，例如播放，录制和暂停，以便于实时控制从服务器到客户端（视频点播）或从客户端到服务器（语音录音）的媒体流。

流数据本身的传输不是RTSP的任务。大多数RTSP服务器使用实时传输协议（RTP）和实时传输控制协议（RTCP）结合媒体流传输    

**Rtcp**   
实时传输控制协议（Real-time Transport Control Protocol或RTP Control Protocol或简写RTCP）是实时传输协议（RTP）的一个姐妹协议。RTCP由RFC 3550定义（取代作废的RFC 1889）。RTP 使用一个 偶数 UDP port ；而RTCP 则使用 RTP 的下一个 port，也就是一个奇数 port。

RTCP为RTP媒体流提供信道外（out-of-band）控制。RTCP本身并不传输数据，但和RTP一起协作将多媒体数据打包和发送。RTCP定期在流多媒体会话参加者之间传输控制数据。RTCP的主要功能是为RTP所提供的服务质量（Quality of Service）提供反馈。

RTCP收集相关媒体连接的统计信息，例如：传输字节数，传输分组数，丢失分组数，jitter，单向和双向网络延迟等等，网络应用程序即可利用RTCP的统计信息来控制传输的品质，比如当网络带宽高负载时限制信息流量或改用压缩比较小的编解码器。   

**总结来说：**      
**实时流协议（setup,play,pause等命令的传输），实时传输协议（内置ts流的数据流），实时传输控制协议（品控）**   
