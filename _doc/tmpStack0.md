
* 利用okhttp提供的certificatePinner类完成证书绑定，防止中间人攻击。


- UIThreadMonitor模块，通过给 Choreographer注册回调，监控应用的帧率
- LooperMonitor模块，Looper传入自定义Printer标记Message处理的开始和结束
- AnrMonitor模块，触发5s延时的任务，监控应用anr
- EvilMethod模块，慢方法监测，监测一个message的处理时间，超过800ms上报信息
- 对易卡顿控件的遍历过程进行 AOP 织入，监控控件遍历耗时
- 对 LayoutInflater 添加自定义的 factory2，xml 加载超时后上报
- jvm 添加 heapdumponoutofmemoryerror 标志，监测到有hprof文件后上报信息
- 构建启动任务的 Pert图，参考阿里的 alpha 框架设计一个异步加载启动器