---
title: 迅雷下载宝OPENWRT官方版本折腾SD卡扩容
date: 2021-09-07 13:32:54
tags: 
	- openwrt
	- xunlei
categories:
  - - Networking
---

# 迅雷下载宝使用OPENWRT 21.02正式版

Breed下直接刷入本来直接可用，没有太多需要折腾的。但是空间不大，而下载宝有个SD卡槽，可以折腾ExtRoot扩容。

[官方文档](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration)

# 一些关键步骤

替换源、安装必要内核模块

我把sd卡分区后格式化成f2fs了，官方里面推荐ext4
```bash

sed -i 's|downloads.openwrt.org|mirrors.aliyun.com/openwrt|' /etc/opkg/distfeeds.conf
sed -i 's|downloads.openwrt.org|mirrors.tencent.com/openwrt|' /etc/opkg/distfeeds.conf


opkg update
opkg install block-mount kmod-fs-f2fs kmod-fs-ext4 kmod-usb-storage kmod-usb-ohci kmod-usb-uhci fdisk kmod-sdhci-mt7620 f2fs-tools f2fsck mkf2fs
```

以下才是关键步骤

```bash

block info
# 查到具体分区的uuid， 对应替换

uci -q delete fstab.rwm
uci set fstab.rwm="mount"
uci set fstab.rwm.device="/dev/mtdblock6"
uci set fstab.rwm.target="/rwm"
uci commit fstab

uci -q delete fstab.overlay
uci set fstab.overlay="mount"
uci set fstab.overlay.uuid="010624d4-e8a2-432c-8fde-b23cf18ebe20"
uci set fstab.overlay.target="/overlay"
uci commit fstab

#迁移数据
mkdir -p /tmp/cproot
mount --bind /overlay /tmp/cproot
mount /dev/mmcblk0p1 /mnt
tar -C /tmp/cproot -cvf - . | tar -C /mnt -xf -	
umount /tmp/cproot /mnt
reboot
```

# 关于升级

升级后这些配置都没了，都得重新搞。sd卡的分区没必要格式化重来，但是以上步骤的数据迁移还是要做，升级后overlay的文件会有更新。内核`/lib/modules`对应版本的目录可以删除。

最重要一点，要删掉`.extroot-uuid`

```
mount /dev/sda1 /mnt
rm -f /mnt/.extroot-uuid /mnt/etc/.extroot-uuid
umount /mnt

```
见官方文档的Troubleshooting  block: extroot: UUID mismatch