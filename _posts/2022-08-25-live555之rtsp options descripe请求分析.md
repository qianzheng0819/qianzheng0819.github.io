---
layout: post
title:  "live555之rtsp options descripe 请求分析"
date:   2022-08-25 11:39:00 +0800
tags:
  - 音视频开发
description:
---

前言
--------
关于rtsp socket如何创建，我不想多说了，无非是bind listen accept三部曲配合live555网络框架的底层select功能，完成数据的收发。

好像大多数服务器都是如此实现，比如我阅读过的python tornado服务器。

OPTIONS请求
------------------
先看下server对该请求的响应报文

{% highlight c %}
RTSP/1.0 200 OK
CSeq: 2
Date: Fri, Jul 08 2022 01:52:56 GMT
Public: OPTIONS, DESCRIBE, SETUP, TEARDOWN, PLAY, PAUSE, GET_PARAMETER, SET_PARAMETER
{% endhighlight %}

再看下相应的处理代码
{% highlight c %}
void RTSPServer::RTSPClientConnection::handleCmd_OPTIONS()
{
  snprintf((char *)fResponseBuffer, sizeof fResponseBuffer,
           "RTSP/1.0 200 OK\r\nCSeq: %s\r\n%sPublic: %s\r\n\r\n",
           fCurrentCSeq, dateHeader(), fOurRTSPServer.allowedCommandNames());
}
{% endhighlight %}

阅读方法名就可以知道，对OPTIONS请求的处理就是返回服务器所支持的方法。

DESCRIPE请求
---------------------
先看下server对该请求的响应报文

{% highlight c %}
RTSP/1.0 200 OK
CSeq: 3
Date: Fri, Jul 08 2022 01:52:56 GMT
Content-Base: rtsp://192.168.43.110:8554/huangh.mkv/
Content-Type: application/sdp
Content-Length: 676

v=0
o=- 1657245176985236 1 IN IP4 192.168.43.110
s=Matroska video+audio+(optional)subtitles, streamed by the LIVE555 Media Server
i=huangh.mkv
t=0 0
a=tool:LIVE555 Streaming Media v2022.06.16
a=type:broadcast
a=control:*
a=range:npt=0-15.438
a=x-qt-text-nam:Matroska video+audio+(optional)subtitles, streamed by the LIVE555 Media Server
a=x-qt-text-inf:huangh.mkv
m=video 0 RTP/AVP 96
c=IN IP4 0.0.0.0
b=AS:500
a=rtpmap:96 H264/90000
a=fmtp:96 packetization-mode=1;profile-level-id=64001E;sprop-parameter-sets=Z2QAHqzZQMg7+IhAAAD6QAAu4APFi2WA,aOvjyyLA
a=control:track1
m=audio 0 RTP/AVP 97
c=IN IP4 0.0.0.0
b=AS:48
a=rtpmap:97 AC3/44100
a=control:track2
{% endhighlight %}

再看下相应的处理代码
{% highlight c %}
void RTSPServer::RTSPClientConnection ::handleCmd_DESCRIBE(char const *urlPreSuffix, char const *urlSuffix, char const *fullRequestStr)
{
  ServerMediaSession *session = NULL;
  char *sdpDescription = NULL;
  char *rtspURL = NULL;
  do
  {
    char urlTotalSuffix[2 * RTSP_PARAM_STRING_MAX];
    // enough space for urlPreSuffix/urlSuffix'\0'
    urlTotalSuffix[0] = '\0';
    if (urlPreSuffix[0] != '\0')
    {
      strcat(urlTotalSuffix, urlPreSuffix);
      strcat(urlTotalSuffix, "/");
    }
    strcat(urlTotalSuffix, urlSuffix);

    if (!authenticationOK("DESCRIBE", urlTotalSuffix, fullRequestStr))
      break;

    // We should really check that the request contains an "Accept:" #####
    // for "application/sdp", because that's what we're sending back #####

    // Begin by looking up the "ServerMediaSession" object for the specified "urlTotalSuffix":
    session = fOurServer.lookupServerMediaSession(urlTotalSuffix);
    if (session == NULL)
    {
      handleCmd_notFound();
      break;
    }

    // Increment the "ServerMediaSession" object's reference count, in case someone removes it
    // while we're using it:
    session->incrementReferenceCount();

    // Then, assemble a SDP description for this session:
    sdpDescription = session->generateSDPDescription();
    if (sdpDescription == NULL)
    {
      // This usually means that a file name that was specified for a
      // "ServerMediaSubsession" does not exist.
      setRTSPResponse("404 File Not Found, Or In Incorrect Format");
      break;
    }
    unsigned sdpDescriptionSize = strlen(sdpDescription);

    // Also, generate our RTSP URL, for the "Content-Base:" header
    // (which is necessary to ensure that the correct URL gets used in subsequent "SETUP" requests).
    rtspURL = fOurRTSPServer.rtspURL(session, fClientInputSocket);

    snprintf((char *)fResponseBuffer, sizeof fResponseBuffer,
             "RTSP/1.0 200 OK\r\nCSeq: %s\r\n"
             "%s"
             "Content-Base: %s/\r\n"
             "Content-Type: application/sdp\r\n"
             "Content-Length: %d\r\n\r\n"
             "%s",
             fCurrentCSeq,
             dateHeader(),
             rtspURL,
             sdpDescriptionSize,
             sdpDescription);
  } while (0);

  if (session != NULL)
  {
    // Decrement its reference count, now that we're done using it:
    session->decrementReferenceCount();
    if (session->referenceCount() == 0 && session->deleteWhenUnreferenced())
    {
      fOurServer.removeServerMediaSession(session);
    }
  }

  delete[] sdpDescription;
  delete[] rtspURL;
}
{% endhighlight %}

对DESCRIPE请求的相应就是返回sdp报文。

#### 什么是sdp
SDP（Session Description Protocol）是一种通用的会话描述协议，主要用来描述多媒体会话，用途包括会话声明、会话邀请、会话初始化等。

参考[会话描述协议SDP](https://zhuanlan.zhihu.com/p/75492311)
