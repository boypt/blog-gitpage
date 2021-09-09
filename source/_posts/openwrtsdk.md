---
title: 使用Docker的Openwrt SDK编译环境
date: 2021-09-09 13:41:11
tags:
    - openwrt
    - docker
    - sdk
categories:
    - Networking
---

Openwrt的编译环境算是比较易用的，只要下载对应设备的sdk包，解压就能使用。

不过一些基础程序包还是依赖原生系统，如果原系统的版本出现冲突，又会有麻烦。不过openwrt官方维护的各种架构和型号的docker镜像，基本都是特定版本的debian，统一化了环境，这就方便了许多。
<!-- more -->

## 例子： 给openwrt 21.02编译一个aria2 1.36

openwrt 21.02 版本时候，aria2 才1.35，所以正式版仓库里面不会有1.36，可以自己编一个。

```
# 拖回一个镜像，这里以一个mtk 7621设备为例子
docker pull openwrtorg/sdk:mipsel_24kc-openwrt-21.02

# 开一个临时镜像一用，在里面编译完就不要了，所以用上了--rm，
# 里面的终端做任何事情只要退了就一切消失
docker run -it --rm openwrtorg/sdk:mipsel_24kc-openwrt-21.02 /bin/bash

# 里面是个debian10的普通用户，但是vim什么的都没有，随便搞一下

sudo sed -i 's|deb.debian.org|mirrors.aliyun.com|;s|security.debian.org|mirrors.aliyun.com|' /etc/apt/sources.list
sudo apt update 

# lrzsz是为了方便跟本地传文件，不然编译完要回去用docker copy命令拷贝多麻烦，
# sz 一下就能把容器里面的文件发到终端了
sudo apt install vim-tiny lrzsz

#普通用户的根目录就是openwrt的编译目录

cp feeds.conf.default feeds.conf
vim feeds.conf
# 复制一份配置, 因为aria2只在packages仓库，因此只保留一行
# src-git packages https://git.openwrt.org/feed/packages.git^65057dcbb5de371503c9159de3d45824bec482e0
# 又因为后面的^65057...是21.02版本的id，我需要最新版的，去掉。
# 就变成 src-git packages https://git.openwrt.org/feed/packages.git

# 从这里开始是需要联网操作的，如果环境在墙内需要设置代理，随便指定个http代理的就好
export http_proxy=http://192.168.0.99:3128

./scripts/feeds update packages
# feeds 命令是同步仓库里面的包源码到本地

./scripts/feeds install aria2
# 因为aria2 1.36已经在仓库里面了，不需要改源码
# 如果仓库没及时更新，可以修改package/feeds/packages/aria2/Makefile

# 直接开片
make package/aria2/compile

# 期间会出现menuconfig，需要找到选择编译模块等等的
# 完成后直接在bin目录里面找到对应的ipk，以及对应的依赖
# 打包后sz出来发到设备里面安装即可
```