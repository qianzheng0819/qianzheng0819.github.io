---
layout: post
title:  "live555之rtsp setup请求分析"
date:   2022-08-26 09:39:00 +0800
tags:
  - 音视频开发
description:
---

SETUP请求分析
----------------
一共收到了两次setup请求，先看下第一次setup的请求报文
{% highlight c %}
SETUP rtsp://192.168.43.110:8554/huangh.mkv/track1 RTSP/1.0
CSeq: 4
User-Agent: LibVLC/3.0.16 (LIVE555 Streaming Media v2016.11.28)
Transport: RTP/AVP;unicast;client_port=43646-43647
{% endhighlight %}

再看下这次setup请求的响应报文
{% highlight c %}
RTSP/1.0 200 OK
CSeq: 4
Date: Fri, Jul 08 2022 01:52:56 GMT
Transport: RTP/AVP;unicast;destination=192.168.43.1;source=192.168.43.110;client_port=43646-43647;server_port=6970-6971
Session: 197DC746;timeout=65
{% endhighlight %}

看一下rtsp server对setup的处理代码

{% highlight cpp %}
void RTSPServer::RTSPClientSession ::handleCmd_SETUP(RTSPServer::RTSPClientConnection *ourClientConnection,
                                                     char const *urlPreSuffix, char const *urlSuffix, char const *fullRequestStr)
{
  do
  {
    // First, make sure the specified stream name exists:
    ServerMediaSession *sms = fOurServer.lookupServerMediaSession(streamName, fOurServerMediaSession == NULL);
    ...
{% endhighlight %}

第一个关键代码是`ServerMediaSession *sms = fOurServer.lookupServerMediaSession(streamName, fOurServerMediaSession == NULL)`

首先要弄清楚fOurServer在运行时实际是什么类，如果直接f4进入会直接跳转GenericMediaServer.cpp,实现为
{% highlight cpp %}
ServerMediaSession* GenericMediaServer
::lookupServerMediaSession(char const* streamName, Boolean /*isFirstLookupInSession*/) {
  // Default implementation:
  return (ServerMediaSession*)(fServerMediaSessions->Lookup(streamName));
}
{% endhighlight %}
fServerMediaSessions是一个hashmap数据结构。这里显然是一个陷阱，是父类方法的默认实现。那么fOurServer到底是什么类呢？

答案在live555MediaServer应用程序的入口方法里
{% highlight cpp %}
int main(int argc, char** argv) {
  ...
  rtspServer = DynamicRTSPServer::createNew(*env, rtspServerPortNum, authDB);
  ...
{% endhighlight %}

fOurServer和rtspServer指向同一对象DynamicRTSPServer.class。下图给出类图关系。
![p]({{ site.baseurl }}/assets/images/2022-pic/p10.png)

{% highlight cpp %}
ServerMediaSession* DynamicRTSPServer
::lookupServerMediaSession(char const* streamName, Boolean isFirstLookupInSession) {
  // First, check whether the specified "streamName" exists as a local file:
  FILE* fid = fopen(streamName, "rb");
  Boolean fileExists = fid != NULL;

  // Next, check whether we already have a "ServerMediaSession" for this file:
  ServerMediaSession* sms = RTSPServer::lookupServerMediaSession(streamName);
  Boolean smsExists = sms != NULL;

  // Handle the four possibilities for "fileExists" and "smsExists":
  if (!fileExists) {
    if (smsExists) {
      // "sms" was created for a file that no longer exists. Remove it:
      removeServerMediaSession(sms);
      sms = NULL;
    }

    return NULL;
  } else {
    printf("smsExists %d\n", smsExists);
    printf("isFirstLookupInSession %d\n", isFirstLookupInSession);
    if (smsExists && isFirstLookupInSession) {
      // Remove the existing "ServerMediaSession" and create a new one, in case the underlying
      // file has changed in some way:
      removeServerMediaSession(sms);
      sms = NULL;
    }

    if (sms == NULL) {
      printf("sms null, create streamName %s\n", streamName);
      sms = createNewSMS(envir(), streamName, fid);
      addServerMediaSession(sms);
    }

    fclose(fid);
    return sms;
  }
}
{% endhighlight %}

先看下函数的入参，streamName就是huangh.mkv,由于`fOurServerMediaSession == NULL`为true，所以`isFirstLookupInSession`为true。
那么第一次setup,实际走的代码分支是
{% highlight cpp %}
if (smsExists && isFirstLookupInSession) {
  // Remove the existing "ServerMediaSession" and create a new one, in case the underlying
  // file has changed in some way:
  removeServerMediaSession(sms);
  sms = NULL;
}

if (sms == NULL) {
  printf("sms null, create streamName %s\n", streamName);
  sms = createNewSMS(envir(), streamName, fid);
  addServerMediaSession(sms);
}
{% endhighlight %}

看一下关键方法createNewSMS()
{% highlight cpp %}
static ServerMediaSession* createNewSMS(UsageEnvironment& env,
					char const* fileName, FILE* /*fid*/) {
  // Use the file name extension to determine the type of "ServerMediaSession":
  char const* extension = strrchr(fileName, '.');
  if (extension == NULL) return NULL;

  ServerMediaSession* sms = NULL;
  Boolean const reuseSource = False;
  if (strcmp(extension, ".aac") == 0) {
    // Assumed to be an AAC Audio (ADTS format) file:
    NEW_SMS("AAC Audio");
    sms->addSubsession(ADTSAudioFileServerMediaSubsession::createNew(env, fileName, reuseSource));
  } else if (strcmp(extension, ".amr") == 0) {
    // Assumed to be an AMR Audio file:
    NEW_SMS("AMR Audio");
    sms->addSubsession(AMRAudioFileServerMediaSubsession::createNew(env, fileName, reuseSource));
  } else if (strcmp(extension, ".ac3") == 0) {
    // Assumed to be an AC-3 Audio file:
    NEW_SMS("AC-3 Audio");
    sms->addSubsession(AC3AudioFileServerMediaSubsession::createNew(env, fileName, reuseSource));
  } else if (strcmp(extension, ".m4e") == 0) {
    // Assumed to be a MPEG-4 Video Elementary Stream file:
    NEW_SMS("MPEG-4 Video");
    sms->addSubsession(MPEG4VideoFileServerMediaSubsession::createNew(env, fileName, reuseSource));
  } else if (strcmp(extension, ".264") == 0) {
    // Assumed to be a H.264 Video Elementary Stream file:
    NEW_SMS("H.264 Video");
    OutPacketBuffer::maxSize = 100000; // allow for some possibly large H.264 frames
    sms->addSubsession(H264VideoFileServerMediaSubsession::createNew(env, fileName, reuseSource));
  }
  ....
  else if (strcmp(extension, ".mkv") == 0 || strcmp(extension, ".webm") == 0) {
   // Assumed to be a Matroska file (note that WebM ('.webm') files are also Matroska files)
   OutPacketBuffer::maxSize = 100000; // allow for some possibly large VP8 or VP9 frames
   NEW_SMS("Matroska video+audio+(optional)subtitles");

   // Create a Matroska file server demultiplexor for the specified file.
   // (We enter the event loop to wait for this to complete.)
   MatroskaDemuxCreationState creationState;
   creationState.watchVariable = 0;
   MatroskaFileServerDemux::createNew(env, fileName, onMatroskaDemuxCreation, &creationState);
   env.taskScheduler().doEventLoop(&creationState.watchVariable);

   ServerMediaSubsession* smss;
   while ((smss = creationState.demux->newServerMediaSubsession()) != NULL) {
     sms->addSubsession(smss);
   }
 }
{% endhighlight %}
可以看到该方法给不同封装格式的媒体文件，指定了自己的处理类。我们是mkv文件，后续的文件解封装大概率与MatroskaFileServerDemux类相关，
这一点要有敏感的意识。当然你也可以播放mp3或264文件，封装格式不同，相应的解析类不同罢了。
现在我们回到最初的`handleCmd_SETUP`方法里。

接下来执行的代码如下
{% highlight cpp %}
else
    {
      printf("fOurServerMediaSession == NULL %d\n", fOurServerMediaSession == NULL);
      if (fOurServerMediaSession == NULL)
      {
        // We're accessing the "ServerMediaSession" for the first time.
        fOurServerMediaSession = sms;
        fOurServerMediaSession->incrementReferenceCount();
      }
      else if (sms != fOurServerMediaSession)
      {
        // The client asked for a stream that's different from the one originally requested for this stream id.  Bad request:
        ourClientConnection->handleCmd_bad();
        break;
      }
    }

    if (fStreamStates == NULL)
    {
      // This is the first "SETUP" for this session.  Set up our array of states for all of this session's subsessions (tracks):
      fNumStreamStates = fOurServerMediaSession->numSubsessions();
      printf("fNumStreamStates： %d\n", fNumStreamStates);
      fStreamStates = new struct streamState[fNumStreamStates];

      ServerMediaSubsessionIterator iter(*fOurServerMediaSession);
      ServerMediaSubsession *subsession;
      for (unsigned i = 0; i < fNumStreamStates; ++i)
      {
        subsession = iter.next();
        fStreamStates[i].subsession = subsession;
        fStreamStates[i].tcpSocketNum = -1;  // for now; may get set for RTP-over-TCP streaming
        fStreamStates[i].streamToken = NULL; // for now; it may be changed by the "getStreamParameters()" call that comes later
      }
    }
{% endhighlight %}

第一次接收setup请求，`fOurServerMediaSession == NULL`为true,`fStreamStates == NULL`也为true。
第二次接收setup请求，都为false。

至于为什么会有两次setup请求，是因为mkv文件有两个track，对应音频和视频。每一个track有自己的ServerMediaSubsession，有自己独自的socket和rtpsink，这些后续内容会讲到，
这里只给读者这样一个结论。  

接着讲述下一个关键代码
{% highlight cpp %}
subsession->getStreamParameters(fOurSessionId, ourClientConnection->fClientAddr.sin_addr.s_addr,
                                    clientRTPPort, clientRTCPPort,
                                    fStreamStates[trackNum].tcpSocketNum, rtpChannelId, rtcpChannelId,
                                    destinationAddress, destinationTTL, fIsMulticast,
                                    serverRTPPort, serverRTCPPort,
                                    fStreamStates[trackNum].streamToken);
...                                  
{% endhighlight %}

这个方法f4进入，会提示有两个实现类，PassiveServerMediaSubsession.cpp和OnDemandServerMediaSubsession.cpp均有实现，
那么还记得之前的MatroskaFileServerDemux吗？看下面的类图，我们就知道选择哪个实现了
![p]({{ site.baseurl }}/assets/images/2022-pic/p11.png)

{% highlight cpp %}
void OnDemandServerMediaSubsession ::getStreamParameters(unsigned clientSessionId,
                                                         netAddressBits clientAddress,
                                                         Port const &clientRTPPort,
                                                         Port const &clientRTCPPort,
                                                         int tcpSocketNum,
                                                         unsigned char rtpChannelId,
                                                         unsigned char rtcpChannelId,
                                                         netAddressBits &destinationAddress,
                                                         u_int8_t & /*destinationTTL*/,
                                                         Boolean &isMulticast,
                                                         Port &serverRTPPort,
                                                         Port &serverRTCPPort,
                                                         void *&streamToken)
{
  ...
  FramedSource *mediaSource = createNewStreamSource(clientSessionId, streamBitrate);
  ...
  else
      {
        // Normal case: We're streaming RTP (over UDP or TCP).  Create a pair of
        // groupsocks (RTP and RTCP), with adjacent port numbers (RTP port number even).
        // (If we're multiplexing RTCP and RTP over the same port number, it can be odd or even.)
        NoReuse dummy(envir()); // ensures that we skip over ports that are already in use
        for (portNumBits serverPortNum = fInitialPortNum;; ++serverPortNum)
        {
          struct in_addr dummyAddr;
          dummyAddr.s_addr = 0;

          serverRTPPort = serverPortNum;
          rtpGroupsock = createGroupsock(dummyAddr, serverRTPPort);
          if (rtpGroupsock->socketNum() < 0)
          {
            delete rtpGroupsock;
            continue; // try again
          }

          if (fMultiplexRTCPWithRTP)
          {
            // Use the RTP 'groupsock' object for RTCP as well:
            serverRTCPPort = serverRTPPort;
            rtcpGroupsock = rtpGroupsock;
          }
          else
          {
            // Create a separate 'groupsock' object (with the next (odd) port number) for RTCP:
            serverRTCPPort = ++serverPortNum;
            rtcpGroupsock = createGroupsock(dummyAddr, serverRTCPPort);
            if (rtcpGroupsock->socketNum() < 0)
            {
              delete rtpGroupsock;
              delete rtcpGroupsock;
              continue; // try again
            }
          }

          break; // success
        }

        unsigned char rtpPayloadType = 96 + trackNumber() - 1; // if dynamic
        rtpSink = createNewRTPSink(rtpGroupsock, rtpPayloadType, mediaSource);
        if (rtpSink != NULL && rtpSink->estimatedBitrate() > 0)
          streamBitrate = rtpSink->estimatedBitrate();
      }
      ...
      // Set up the state of the stream.  The stream will get started later:
    streamToken = fLastStreamToken = new StreamState(*this, serverRTPPort, serverRTCPPort, rtpSink, udpSink,
                                                     streamBitrate, mediaSource,
                                                     rtpGroupsock, rtcpGroupsock);
}
{% endhighlight %}
首先是`FramedSource *mediaSource = createNewStreamSource(clientSessionId, streamBitrate);`
{% highlight cpp %}
FramedSource* MatroskaFileServerMediaSubsession
::createNewStreamSource(unsigned clientSessionId, unsigned& estBitrate) {
  FramedSource* baseSource = fOurDemux.newDemuxedTrack(clientSessionId, fTrack->trackNumber);
  if (baseSource == NULL) return NULL;

  return fOurDemux.ourMatroskaFile()
    ->createSourceForStreaming(baseSource, fTrack->trackNumber,
			       estBitrate, fNumFiltersInFrontOfTrack);
}
{% endhighlight %}
根据不同的track返回不同的mediaSource，音视频各一个

接着创建rtpGroupsock，rtcpGroupsock,rtcp端口号是rtp端口号加一

最后创建`rtpSink = createNewRTPSink(rtpGroupsock, rtpPayloadType, mediaSource);`
{% highlight cpp %}
RTPSink *MatroskaFile ::createRTPSinkForTrackNumber(unsigned trackNumber, Groupsock *rtpGroupsock,
                                                    unsigned char rtpPayloadTypeIfDynamic)
{
  RTPSink *result = NULL; // default value, if an error occurs

  do
  {
    MatroskaTrack *track = lookup(trackNumber);
    if (track == NULL)
      break;
  ....
    else if (strcmp(track->mimeType, "audio/AC3") == 0)
      {
        result = AC3AudioRTPSink ::createNew(envir(), rtpGroupsock, rtpPayloadTypeIfDynamic, track->samplingFrequency);
      }
    else if (strcmp(track->mimeType, "video/H264") == 0)
    {
      ....
      result = H264VideoRTPSink::createNew(envir(), rtpGroupsock, rtpPayloadTypeIfDynamic,
                                           SPS, SPSSize, PPS, PPSSize);

{% endhighlight %}

我的huang.mkv文件音频编码是AC3,视频编码是H264。所以对应上述代码里的两个rtpsink。这个对于后续我们分析数据的传递很重要。
记住视频track是用H264VideoRTPSink类。

`streamToken = new StreamState(...)`则是把相关的类都封装到这个token里。

完结
----------------
分析到这里，对于SETUP请求的处理基本讲解完了，接下来只要返回响应报文即可。
