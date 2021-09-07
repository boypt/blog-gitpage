---
title: 大麦DW33D刷机总结帖
date: 2021-09-07 13:44:04
tags:
	- wifi
	- rouer
	- dw33d
categories:
  - - Networking
---

大麦DW33D已经是很多年前的路由，都2021了可能配置上的有点过时（1750MAC Wifi+全千兆+高通QCA9558），但是如果本着性价比出发的话，目前某鱼上不到50一台的性价比确实还行的。

<!-- more -->

总结一下刷机要点。

# 背景

dw33d内部有三个存储空间（可理解为硬盘），SPI-NOR(16M)、 NAND(128M)、 TF卡（16G）。

原厂固件是在NOR上的，一些旧版（lede 17.x）也是设计刷到NOR上，这类固件称为(ath1x)。BREED默认也是刷到NOR，启动也是NOR。

后来openwrt把dw33d纳入官方支持时候，改成使用NAND作为固件区域，并称为`ath79`，或者nand固件。

旧版固件虽然刷入简单（可以Breed WEB页面直刷），但是可用空间非常有限。

nand固件的刷入较为复杂，但是有足够大的存储空间（可用空间70M+），可以安装很多可选插件。

Breed虽然在dw33d上工作不完美，但是还是比u-boot简化一丢丢。

刷机方法可以是u-boot，连接TTL线操作；也可以不拆外壳，telnet操作；


# 简洁刷机

Breed刷Nand看起来很复杂，其实总共3个步骤：

- Breed设置环境变量
- PC开启http服务器
- Breed Telnet下载并写入固件。

## 设置环境变量

目的是让breed默认从nand开始启动，`bank 0`是nand空间。


```
envconf 0x6000000 0x20000
env set network.ipaddr 192.168.1.1
env set network.netmask 255.255.255.0
env set autoboot.disabled 0
env set autoboot.delay 5
env set autoboot.command "boot flash bank 0 0x0"
env save

```
## PC上开启HTTP服务器

确保dw33d的地址能访问到pc, 例子中pc的地址是192.168.0.254

## Breed 下载固件


此处原理是，breed命令的wget下载指定地址的文件，放到地址0x80000000。
然后需要注意提示下载的长度(0xd00000)

然后擦除对应长度的nand空间，然后从0x80000000复制写入特定长度的数据

```
wget http://192.168.0.254/firmware/dw33d-factory.bin
##### 注意长度的0xd00000
#####-> Length: 13631488/0xd00000 (13MB) [application/octet-stream]
flash bank 0 erase 0x0 0xd00000
flash bank 0 write 0x0 0x80000000 0xd00000
```

# 关于升级

测试过从21.02-rc升级到21.02正式版，直接web升级没出现问题。

但是软件包需要重装，常用工具curl/tcpdump之类。
