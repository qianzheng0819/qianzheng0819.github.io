1.introduce mirasink program
----------------------------

- ndk开发，cmake构建项目
- p2p组网，传递ip和端口
- miracast协议六步握手
- rtp包解析，获取播放信息和ts流payload
- rtp包根据cSeq重新排序，以及对延时包的处理
- 获取到ts裸流后，根据流类型回传数据，播放器播放
- 使用io多路复用技术，管理rtsp和rtcp对应socket的数据收发
- rtcp包回传RR（Receiver Report）包,向send端回馈延时情况

2.miracast主流程
---------------------
- rtp包的缓冲优先队列，允许包超时时间为100ms。如果期望包未到达，缓冲包到队列里。
  如果超时，则期望包序列号加一。继续这个循环。

- rtp包解析完成获取ts流payload。ts流的payload分两种类型，psi和pes。ts流还自带适配域，
  一个目的是填充188字节ts流，第二个目的则是每100ms传递PCR(program clock ref)给客户端的解码器矫正STC(system timing clock),pcr是参考90khz时钟。

- psi的解析，首先拿到pid固定为0x00的PAT表，通过pat表获取到pmt表，pat是一种电视广播系统里的
  节目表。我们只用关注自身需要的节目，通过节目映射到pmt表，再解析pmt表获取到当前节目里的流
  （一般为视频和音频两种数据流）

- 以视频流举例，拿到流pid后，我们可以获取视频流的ts包。一般视频帧的大小是2k字节左右，那么一个
  ts包显然不能完成一帧。这个时候要去完成pes拼包的操作，依靠的是Payload Unit Start Indicator
  它为1时代表一个完整帧的开始。那么我们遇到下一个指示符时，就可以把之前拼接的包flush出去。

- 接着就是pes包的解析，重点是解析出pts和dts。解析后获得的payload则是我们期望的h264裸流。
  裸流数据携带pts转化的时间戳，和流数据类型。把获取的裸流抛到jni层，即可完成显示。
