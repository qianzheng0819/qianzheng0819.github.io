---
layout: post
title:  "live555之rtsp play请求分析"
date:   2022-08-26 15:39:00 +0800
tags:
  - 音视频开发
description:
---

PLAY请求分析
---------------
PLAY请求的相关处理比较简单，就是触发推流。难点是流数据是如何流动起来的，怎么从媒体文件解封装，解码，再封包发送。

首先看下play请求报文
{% highlight cpp %}
PLAY rtsp://192.168.43.110:8554/huangh.mkv/ RTSP/1.0
CSeq: 6
User-Agent: LibVLC/3.0.16 (LIVE555 Streaming Media v2016.11.28)
Session: 197DC746
Range: npt=0.000-
{% endhighlight %}

rtspServer是一个一对多的服务器，Session id用来标记相应的session

接着看下代码是如何响应play请求吧
{% highlight cpp %}
void RTSPServer::RTSPClientSession ::handleCmd_PLAY(RTSPServer::RTSPClientConnection *ourClientConnection,
                                                    ServerMediaSubsession *subsession, char const *fullRequestStr)
{
  .....
  // 处理range,scale
  .....
  for (i = 0; i < fNumStreamStates; ++i)
   {
     if (subsession == NULL /* means: aggregated operation */
         || subsession == fStreamStates[i].subsession)
     {
       unsigned short rtpSeqNum = 0;
       unsigned rtpTimestamp = 0;
       if (fStreamStates[i].subsession == NULL)
         continue;
       fStreamStates[i].subsession->startStream(fOurSessionId,
                                                fStreamStates[i].streamToken,
                                                (TaskFunc *)noteClientLiveness, this,
                                                rtpSeqNum, rtpTimestamp,
                                                RTSPServer::RTSPClientConnection::handleAlternativeRequestByte, ourClientConnection);
       const char *urlSuffix = fStreamStates[i].subsession->trackId();
       char *prevRTPInfo = rtpInfo;
       unsigned rtpInfoSize = rtpInfoFmtSize + strlen(prevRTPInfo) + 1 + rtspURLSize + strlen(urlSuffix) + 5 /*max unsigned short len*/
                              + 10                                                                           /*max unsigned (32-bit) len*/
                              + 2 /*allows for trailing \r\n at final end of string*/;
       rtpInfo = new char[rtpInfoSize];
       sprintf(rtpInfo, rtpInfoFmt,
               prevRTPInfo,
               numRTPInfoItems++ == 0 ? "" : ",",
               rtspURL, urlSuffix,
               rtpSeqNum,
               rtpTimestamp);
       delete[] prevRTPInfo;
     }
   }
}
{% endhighlight %}

play响应方法很长，但大多数代码是用于处理range(从哪里播放)和scale(播放倍速)，这些东西我们不用太多关注。

核心的代码只有一句`fStreamStates[i].subsession->startStream(...)`，这里音视频两个subsession分别推流。

进入startStream()方法
{% highlight cpp %}
void OnDemandServerMediaSubsession::startStream(unsigned clientSessionId,
                                                void *streamToken,
                                                TaskFunc *rtcpRRHandler,
                                                void *rtcpRRHandlerClientData,
                                                unsigned short &rtpSeqNum,
                                                unsigned &rtpTimestamp,
                                                ServerRequestAlternativeByteHandler *serverRequestAlternativeByteHandler,
                                                void *serverRequestAlternativeByteHandlerClientData)
{
  StreamState *streamState = (StreamState *)streamToken;
  Destinations *destinations = (Destinations *)(fDestinationsHashTable->Lookup((char const *)clientSessionId));
  if (streamState != NULL)
  {
    streamState->startPlaying(destinations, clientSessionId,
                              rtcpRRHandler, rtcpRRHandlerClientData,
                              serverRequestAlternativeByteHandler, serverRequestAlternativeByteHandlerClientData);
    RTPSink *rtpSink = streamState->rtpSink(); // alias
    if (rtpSink != NULL)
    {
      rtpSeqNum = rtpSink->currentSeqNo();
      rtpTimestamp = rtpSink->presetNextTimestamp();
    }
  }
}
{% endhighlight %}

还记得上篇文章中的streamToken吗，它封装了rtpsink,rtpsocket，mediaSource等信息。看下streamState->startPlaying方法

{% highlight cpp %}
void StreamState ::startPlaying(Destinations *dests, unsigned clientSessionId,
                                TaskFunc *rtcpRRHandler, void *rtcpRRHandlerClientData,
                                ServerRequestAlternativeByteHandler *serverRequestAlternativeByteHandler,
                                void *serverRequestAlternativeByteHandlerClientData)
{
  ...
  fRTPSink->startPlaying(*fMediaSource, afterPlayingStreamState, this);
}
{% endhighlight %}

{% highlight cpp %}
Boolean MediaSink::startPlaying(MediaSource& source,
				afterPlayingFunc* afterFunc,
				void* afterClientData) {
  // Make sure we're not already being played:
  if (fSource != NULL) {
    envir().setResultMsg("This sink is already being played");
    return False;
  }

  // Make sure our source is compatible:
  if (!sourceIsCompatibleWithUs(source)) {
    envir().setResultMsg("MediaSink::startPlaying(): source is not compatible!");
    return False;
  }
  fSource = (FramedSource*)&source;

  fAfterFunc = afterFunc;
  fAfterClientData = afterClientData;
  return continuePlaying();
}
{% endhighlight %}

记住MediaSink fSource这个成员变量，它标记了当前的媒体文件source。

下图为RTPSinK的类图关系
![p]({{ site.baseurl }}/assets/images/2022-pic/p12.png)

下图为startStream的流程图
![p]({{ site.baseurl }}/assets/images/2022-pic/p13.png)

看下H264or5VideoRTPSink::continuePlaying()方法
{% highlight cpp %}
Boolean H264or5VideoRTPSink::continuePlaying()
{
  // First, check whether we have a 'fragmenter' class set up yet.
  // If not, create it now:
  if (fOurFragmenter == NULL)
  {
    fOurFragmenter = new H264or5Fragmenter(fHNumber, envir(), fSource, OutPacketBuffer::maxSize,
                                           ourMaxPacketSize() - 12 /*RTP hdr size*/);
  }
  else
  {
    fOurFragmenter->reassignInputSource(fSource);
  }
  fSource = fOurFragmenter;

  // Then call the parent class's implementation:
  return MultiFramedRTPSink::continuePlaying();
}
{% endhighlight %}
H264or5Fragmenter是H264or5VideoRTPSink.cpp的内部类。H264or5Fragmenter作用是为h264帧作分包。普通的视频帧大小往往超过10kB，而以太网的MTU一般为1500B，
所以视频帧发送时需要分包操作。

关于后续的方法调用，我想用gdb调试获取的方法栈向大家展示
![p]({{ site.baseurl }}/assets/images/2022-pic/p14.png)

数据的大概流向是：

startStream触发MultiFramedRTPSink.sendNext,然后触发H264or5Fragmenter::doGetNextFrame()，接着通过MatroskaFile获取解封装后的数据，数据回传并
触发H264or5Fragmenter.afterGettingFrame,然后分包后的数据一个个回传给MultiFramedRTPSink，并sink出去。然后获取下一帧，循环反复。

下面贴一下关键代码

{% highlight cpp %}
void H264or5Fragmenter::doGetNextFrame()
{
  if (fNumValidDataBytes == 1)
  {
    // 获取下一帧
    // We have no NAL unit data currently in the buffer.  Read a new one:
    fInputSource->getNextFrame(&fInputBuffer[1], fInputBufferSize - 1,
                               afterGettingFrame, this,
                               FramedSource::handleClosure, this);
  }
  else
  {
    // 视频帧分包
    // We have NAL unit data in the buffer.  There are three cases to consider:
    // 1. There is a new NAL unit in the buffer, and it's small enough to deliver
    //    to the RTP sink (as is).
    // 2. There is a new NAL unit in the buffer, but it's too large to deliver to
    //    the RTP sink in its entirety.  Deliver the first fragment of this data,
    //    as a FU packet, with one extra preceding header byte (for the "FU header").
    // 3. There is a NAL unit in the buffer, and we've already delivered some
    //    fragment(s) of this.  Deliver the next fragment of this data,
    //    as a FU packet, with two (H.264) or three (H.265) extra preceding header bytes
    //    (for the "NAL header" and the "FU header").

    if (fMaxSize < fMaxOutputPacketSize)
    { // shouldn't happen
      envir() << "H264or5Fragmenter::doGetNextFrame(): fMaxSize ("
              << fMaxSize << ") is smaller than expected\n";
    }
    else
    {
      fMaxSize = fMaxOutputPacketSize;
    }
    ......
  }
{% endhighlight %}
fInputBuffer[]用来缓存视频帧数据的，其分配大小是60KB。可以把它认为是数据传递的source。

{% highlight cpp %}
void MultiFramedRTPSink::packFrame()
{
  // Get the next frame.

  // First, skip over the space we'll use for any frame-specific header:
  fCurFrameSpecificHeaderPosition = fOutBuf->curPacketSize();
  fCurFrameSpecificHeaderSize = frameSpecificHeaderSize();
  fOutBuf->skipBytes(fCurFrameSpecificHeaderSize);
  fTotalFrameSpecificHeaderSizes += fCurFrameSpecificHeaderSize;

  // See if we have an overflow frame that was too big for the last pkt
  if (fOutBuf->haveOverflowData())
  {
    // Use this frame before reading a new one from the source
    unsigned frameSize = fOutBuf->overflowDataSize();
    struct timeval presentationTime = fOutBuf->overflowPresentationTime();
    unsigned durationInMicroseconds = fOutBuf->overflowDurationInMicroseconds();
    fOutBuf->useOverflowData();

    afterGettingFrame1(frameSize, 0, presentationTime, durationInMicroseconds);
  }
  else
  {
    // Normal case: we need to read a new frame from the source
    if (fSource == NULL)
      return;
    fSource->getNextFrame(fOutBuf->curPtr(), fOutBuf->totalBytesAvailable(),
                          afterGettingFrame, this, ourHandleClosure, this);
  }
}
{% endhighlight %}
fOutBuf就是FramedSource里的fTo,可以把它认为是数据传递的dest。

结语
--------------
视频数据的传递就此结束了，其中的细节你可以根据我提供的方法栈和关键方法去摸索。我这里只能说一个大概的结论，其中的细节只能自己体会。

至于音频数据的传递基本类似视频数据，只是少了H264or5Fragmenter这一层封装，直接把一个音频帧传递出去就行。
