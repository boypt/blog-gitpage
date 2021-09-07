---
title: auhome-spot-cube
date: 2021-09-07 10:34:27
tags: 
	- wifi 
	- router
	- ‎PXH11RWA
---

# 日本 au home spot cube 水晶盒子Wifi路由


## 原厂固件5G加入中国频段方法

  * 用户名`root`
  * 密码`plumeria0077`

打开`192.168.0.1/syscmd.asp`

输入命令：

```
flash sethw HW_WLAN1_REG_DOMAIN 2
flash sethw HW_WLAN0_REG_DOMAIN 2
```

重启后进入设置，打开SSID3设置页面(确保5G必须打开)，接着进入`192.168.0.1/wlbasic.asp`选频段，默认是自动，可以设置149和153两个国内信道。
再次重启就可以了。

[原文地址](http://www.right.com.cn/forum/thread-164531-1-1.html)

## 1、关闭WPS

2.4Ghz
```
flash set WLAN0_WSC_DISABLE 1
```
5Ghz
```
flash set WLAN1_WSC_DISABLE 1
```
害怕被PIN可以看一下这些文字：
If the wrong PIN code three times, You will not be able to connect more. Please press the button below to release it.      
UnLock
我也曾mdk3攻击路由器，使它重启，但是攻击5分钟无效。

==== 2、使用中国5Ghz频道 ====

```
flash set WLAN0_CHANNEL 149
```

==== 3、更改时区为东八区 ====

```
flash set NTP_TIMEZONE -8\ 1
```

==== 4、更改LAN IP地址 ====

```
flash set IP_ADDR 192.168.0.1
flash set SUBNET_MASK 255.255.255.0
flash set DHCP_CLIENT_START 192.168.0.100
flash set DHCP_CLIENT_END 192.168.0.200
```

==== 5、更改用户名密码 ====

```
flash set SUPER_NAME root
flash set SUPER_PASSWORD plumeria0077
flash set USER_NAME au
flash set USER_PASSWORD 1234
```

==== 6、修改PIN码 ( 此项修改之后是初始化无法复原的 ) ====

2.4Ghz
```
flash sethw HW_WLAN0_WSC_PIN xxxxxxxx
```
5Ghz
```
flash sethw HW_WLAN1_WSC_PIN xxxxxxxx
```

===== sdk固件要刷回原版，可以参考【N500R_TTL+TFTP写入教程】， =====

注意：

1. 加电时RESET， 然后192.168.1.6 就可以了，电脑地址要设为192.168.1.x。
2. tftp上传的原版固件威编程器固件，大小是8M的。8192kB3. 


刷机命令


`FLW 0 80500000 800000`


注意一定是大写的。
4. 启动后，登陆时账号是root，密码plumeria0077，你们的可以试试哈。

5. 关于5G 149信道设置：虽然设置为149，但在【本机状态】5G信道是36，解决方法：

打开【系统维护】【系统命令行】或者192.168.0.1/syscmd.asp
输入命令：
```
flash sethw HW_WLAN1_REG_DOMAIN 2
flash sethw HW_WLAN0_REG_DOMAIN 2
```

重启后进入设置，打开SSID3设置页面(确保5G必须打开)，接着进入【无线设置】或192.168.0.1/wlbasic.asp选频段，默认是自动，
可以设置149和153两个国内信道。新版的信道可以输入。
再次重启就可以了。
可以在【本机状态】查看5G是否是149信道。

# 恢复Ccalibration Power


赶紧查了一下，找到了[Embedded System] 手動設定 Ccalibration Power 搞定。

怎么样修改Ccalibration Power呢，举个栗子，说一下。
1、

```
HW_WLAN0_TX_POWER_CCK_A=3333333131313131313131313131
```

33是16进制转换10进制为51，31转换为49。

命令是：

```
flash set HW_WLAN0_TX_POWER_CCK_A 51 51 51 49 49 49 49 49 49 49 49 49 49 49
```

2、复杂一点的：
```
HW_WLAN0_TX_POWER_5G_HT40_1S_A=00000000000000000000000000000000000000000000000000000000000000000000002f2f2f2f2f2f2f2f2f2f2d2d2d2d2d2d2d2d2d2d2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2c2c2c2c2c2c2d2d2d2d2d2d000000000000000000000000000000000000000000000000000000000000
```

你自己转换分组的，我累了。（末尾的00舍弃）

```
flash sethw HW_WLAN0_TX_POWER_5G_HT40_1S_B 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 47 47 47 47 47 47 47 47 47 47 45 45 45 45 45 45 45 45 45 45 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 43 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 44 44 44 44 44 44 45 45 45 45 45 45
```