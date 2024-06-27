

# chcp 65001

# git push origin HEAD:refs/for/master

# /system/priv-app/VCOS_VoyahMusic

# 3.15 版本亮屏
    adb root
    adb remount
    adb shell
    telnet cdc-qnx
    root
    megapm_switch_state.sh
    选 1 ，然后选 4

    10.195.0.74


# U盘模式的切换：
adb root
adb remount
adb shell
setenforce 0

（1）setprop persist.vendor.usb.mode device (adb 调试模式 )
（2）setprop persist.vendor.usb.mode host（U盘模式）

# 无线 adb :

(0) 车机电脑在同一网络里面
(1) 查看车机设备ip    adb shell ifconfig wlan0
(2) 设备打开6666端口监听 adb tcpip 6666
(3) 数据线断开，执行命令 adb disconnect（目的是彻底断开刚才的有线连接，以便进行无线连接）；
(4) 连接指定ip地址的设备adb connect 172.20.10.3:6666

systemProp.http.proxyHost=127.0.0.1
systemProp.http.proxyPort=10809
systemProp.https.proxyHost=127.0.0.1
systemProp.https.proxyPort=10809
