---
title: 树莓派使用硬件RTC时钟
date: 2021-10-22 13:19:03
tags: 
	- linux
	- raspberry
categories:
	- linux
---

树莓派等各种派，往往都不带硬件时钟，在断电、离线网络等情况下是无法以实际时间运行系统和程序的。解决方式是安装一个RTC时钟模块，成本也不高，但是软硬件各方面可能有一些坑要踩一下。

![DS1307模块](ds1307.png)

<!-- more -->

## 硬件模块

硬件上可以选择上图这种专门为raspberry 40pin 接口使用的小模块，成本较高（淘宝售价10元+），但是组装性能好一些，跟主板本身能紧密连接。性能方面，ds1307其实只是个计数芯片，实际走时是通过外部32768Hz的晶振，精度可能不会十分准确。另外这个模块扣具一下子占了10个pin口，其中包括模块通信使用的i2c、电源5v/3.3v，以及常用调试的uart ttl口，不过也引出了插针，调试倒不成问题。

有性能更好的ds3213模块，芯片内部自带晶振，受干扰和晶振质量影响较小，走时较为准确，不过成本高一些。

![DS3213模块](ds3213.png)

如果使用更通用的模块，ds1307甚至有卖2块钱的小模块，只是需要另外连接引线。其实这小小模块的性能考虑还挺多的，比如有些模块提供电池充电等等。


## 软件配置

上述这些RTC模块其实使用了i2c通信接口，而树莓派对应的系统基本都要对应的内核模块支持，要驱动起来不算麻烦。

比如在raspberry 4B，只需要配置`/boot/config.txt` (具体文档在`/boot/overlays/README`)：

```
dtoverlay=i2c-rtc,ds1307 
dtparam=i2c=on
```

具体不通型号稍有区别，可以看[51pi](https://wiki.52pi.com/index.php/DS1307_RTC_Module_with_BAT_for_Raspberry_Pi_SKU%3A_EP-0059)。

但是根据使用系统的不通，发现并不好用，虽然驱动加上了，系统时间并没有从硬件同步进来（也许是centos系列系统的问题？包括树莓自己的Raspberry Pi OS）。

查到一个方法是写自己的systemd服务，脚本控制模块加载和同步时间。

[addon-hwclock.service](https://github.com/scottlamb/moonfire-nvr/wiki/System-setup#realtime-clock-on-raspberry-pi)


对应的脚本和service如下：

脚本 `/etc/addon-hwclock`


```bash
#!/bin/bash
# Use add-on hardware RTC module. See /etc/systemd/system/addon-hwclock.service.

# Be verbose so failures can be examined with `journalctl --unit addon-hwclock`.
set -o errexit
set -o xtrace

if [[ "$1" = "start" ]]; then
    # Load the kernel modules needed by the RTC.
    # udevd might do some/all of this later, but we want it now (before root fsck).
    for i in i2c_bcm2835 rtc_ds1307; do /sbin/modprobe $i; done

    # Potentially useful debugging commands. Uncomment to taste.
    # /usr/bin/dtc --in-format fs /proc/device-tree
    # /usr/sbin/i2cdetect -y 1

    # echo ds3231 0x68 > /sys/class/i2c-adapter/i2c-1/new_device
    echo ds1307 0x68 > /sys/class/i2c-adapter/i2c-1/new_device

    # Debugging, again.
    # /usr/bin/dtc --in-format fs /proc/device-tree
    # /usr/sbin/i2cdetect -y 1

    /sbin/hwclock --hctosys --utc

elif [[ "$1" = "stop" ]]; then
    /sbin/hwclock --systohc --utc
fi
```

Service `/etc/systemd/system/addon-hwclock.service`

```
[Unit]
Description=Add-on hardware RTC module
DefaultDependencies=no

# Run after fake-hwclock so it doesn't override the time set from the real hwclock.
After=fake-hwclock.service

# Filesystems record their last mount time; fsck complains if it's in the future.
# Run before the first fsck to avoid this. Earlier in the boot process is better anyway.
Before=sysinit.target systemd-fsck-root.service time-set.target
Conflicts=shutdown.target

# Note because of the WantedBy=sysinit.target, the system will boot into
# emergency mode on failure. Be a little defensive with the ConditionFileIsExecutable
# to avoid that annoying failure mode.
ConditionFileIsExecutable=/etc/addon-hwclock

[Service]
Type=oneshot
RemainAfterExit=yes

ExecStart=/etc/addon-hwclock start
ExecStop=/etc/addon-hwclock stop

[Install]
WantedBy=sysinit.target

```

然后激活使用，把当前时间写入到模块（--systohc）

```bash
sudo chmod a+rx /etc/addon-hwclock
sudo systemctl enable addon-hwclock.service


sh -c 'for i in i2c_bcm2835 rtc_ds1307; do /sbin/modprobe $i; done'
sh -c 'echo ds1307 0x68 > /sys/class/i2c-adapter/i2c-1/new_device'
/sbin/hwclock --systohc --utc
```

然后就可以重启检查效果了。

## 跳坑

### 电源稳定性

发现虽然这么个小模块，也是对电源有要求的，使用主机USB、1A小电源启动树莓派后，会间歇性找不到RTC模块，或者各种其他报错。

换成5V 2A的电源就稳定了。折腾了不少时间。

### 调试

i2c是个总线接口，可以连接很多设备，分别使用不同的地址进行通信。

使用`i2cdetect -y 1`可以扫描检测所有i2c设备。

比如ds1307模块使用的地址是0x68，上述扫描时候如果发现对应地址有设备就会显示对应的数字。

但是如果已经加载好驱动模块，对应地址就显示`UU`。

### 关于fake-hwclock服务

fake-hwclock其实是树莓派系统默认对于不存在硬件时钟时候，把关机时候的时间写入到文件，开机时候重新读取，而不是从1970年开始重新算的时间。

不少资料说使用rtc模块后应该删除这个服务，其实不必须的，不影响上述模块的工作流程。还能避免在临时断开模块时候设备启动的时间又从1970开始的问题。 

### 硬件连线

使用硬件连线模块时候，注意是ds1307用的是5v，DS3231 和 PCF8523是用3.3v的。

DS3231 和 PCF8523 使用3.3v供电。
** Vin 连接至 Pin 1 **
** SDA 连接到 Pin 3 **
** SCL 连接到 Pin 5 **
** GND 连接到引脚 6 **

![DS3231 3.3v](3v3.png)

DS1307 使用5v供电。

![DS1307 5V](5v.png)

** Vin 连接至 Pin 4 **
** SDA 连接到 Pin 3 **
** SCL 连接到 Pin 5 **
** GND 连接到引脚 6 **
