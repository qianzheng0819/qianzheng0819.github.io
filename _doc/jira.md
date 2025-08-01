
###### 响应时间 #####

# https://tbjira.lenovo.com/issue/browse/DEPLOYT-17563  打开设置（冷启动）响应时间超出标准值150ms

# https://tbjira.lenovo.com/issue/browse/DEPLOYV-4125 GoogleMaps点back键退出时应用图标会覆盖屏幕（应用窗口透明，露出桌面的图标）

# https://tbjira.lenovo.com/issue/browse/DEPLOYV-5880 低电量下手势导航切换至虚拟键导航,按home键返回至主桌面中度卡顿

  石小房关于launcher和sf抢占gpu的相关分析

# https://tbjira.lenovo.com/issue/browse/TENET-10090  测试机打开WhatsApp Messenger的时间比对比机响应时间长，测试机平均值：835.83ms，对比机平均值：575.00ms

  黄宇评论里有远程动画，应用窗口绘制完的相关日志

# https://tbjira.lenovo.com/issue/browse/DEPLOYV-3590

  gms小部件高度3*2，空间不足被截断

# https://tbjira.lenovo.com/issue/browse/DEPLOYV-9177
  http://tbjira.lenovo.com:8081/kb/pages/viewpage.action?pageId=151847232

  打开采用简单动效

# https://tbjira.lenovo.com/issue/browse/DEPLOYV-7055

  项目方提频处理

# https://tbjira.lenovo.com/issue/browse/DEPLOYV-11862

  launcher和sf抢占gpu的相关分析，卡顿

# https://tbjira.lenovo.com/issue/browse/DEPLOYV-8354
# https://tbjira.lenovo.com/issue/browse/TENET-14647
  
  app和浮窗叠加类问题

# https://tbjira.lenovo.com/issue/browse/DEPLOYV-11949

  第三方文件app动画问题

# https://tbjira.lenovo.com/issue/browse/TENET-8753   http://appcode.slab.lenovo.com:8080/#/c/222248/

  时钟图标先变大后正常显示

# getWorkspaceScrimColor  BackgroundAppState

# http://appcode.slab.lenovo.com:8080/#/c/230013/  topaz项目取图标，直接使用btvDrawable

# http://appcode.slab.lenovo.com:8080/#/c/229937/ cava back animation

# https://tbjira.lenovo.com/issue/browse/DEPLOYV-13209 【项目部署-CAVAV】[ST][WIFI ROW][系统-launcher]launcher退出时图标提前显示[2/2台,20/20次]	

# https://tbjira.lenovo.com/issue/browse/LGSIV-13226             wps应用打断动画存在问题

# https://tbjira.lenovo.com/issue/browse/DEPLOYV-7732 动画效果需要优化，有时候发现点击APP时，桌面的图标先变小再打开，整体不够流畅。

  图标点击后回弹慢


# https://tbjira.lenovo.com/issue/browse/LSSIB-8292 退出应用-点击Home键响应时延慢于竞品机

# https://tbjira.lenovo.com/issue/browse/LSSIB-8300  退出应用-点击home键响应时延慢于竞品机
 
 目前从trace日志分析来看，由于接口eglSwapBuffersWithDamageKHR在测试机上耗时异常，会对测试结果产生较大影响。该接口属于openl gl驱动部分，需要vendor做进一步分析。




