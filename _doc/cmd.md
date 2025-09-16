
# fastboot erase system
  fastboot flash system system.img

# adb reboot fastboot
  fastboot flash system *.img
  烧写完成后，fastboot -w  reboot
  用于擦除（wipe）设备的用户数据分区（/data 分区）和缓存分区（/cache 分区）。通常用于在刷写 ROM 或者恢复出厂设置后清除旧数据，以避免潜在的兼容性问题。


# Lapis
  adb reboot bootloader  
  fastboot erase xbl 
  fastboot  reboot

# Kirby and new Lapis
  adb reboot edl

# inception刷机
  downloadXml选择对应的flash.xml文件
  拔usb线，让平板关机
  点击download或者format all + download后连接usb线

# RD-Test
  Lenovo2019
  ####5993#
  adb shell getprop persist.sys.currentversion

# oem解锁
  http://zwork.lenovo.com.cn:8090/pages/viewpage.action?pageId=37860169
  按音量下+电源进 bootloader； 
  在这个页面上输入 fastboot device 显示的设备id 输入到这个页面的 http://ci.slab.lenovo.com:8080/view/Sign/job/Android_oem_unlock/ 的 SN 中生成 unlock  image;
  下载 unlock  image；执行命令 fastboot flash unlock 下载的 unlock image
  设备重启后可能无法开机，按音量下 + 电源经 bootloader 模式，执行 fastboot erase userdata, fastboot erase metadata, fastboot erase misc ， fastboot reboot 重启设备
  执行正常 remount 过程


# adb pull /sdcard/Pictures/screenrecorder
  adb pull /sdcard/log
  adb pull /data/local/traces
  adb pull /system/media/zui_launcher_config.xml
  adb shell am force-stop com.zui.launcher

# TouchInteraction|AbsSwip|QuickstepTransitionManager|StateManager|Animator|ViewRootImpl
  
# 7.23日前gpt续费

# 66428
  

