
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

# Kirby
  adb reboot edl

# RD-Test
  Lenovo2019

