---
title: 在路由器iptables中匹配IPv6动态地址
tags:
  - address
  - iptables
  - ipv6
  - suffix
id: '1421'
categories:
  - - Networking
date: 2018-10-24 15:59:18
---

家用宽带目前很多都部署了IPv6，家用路由器目前Padavan/Openwrt等系统都能较好地支持了IPv6。不过要充分利用IPv6链接设备，有些坑。

<!-- more -->
### 动态变化的IPv6地址

首先是IPv6地址，不同设备（操作系统）获取的IPv6地址有区别，较为通用的是【无状态EUI-64地址】，操作系统通过网卡的mac地址生成一个64位固定后缀，以及路由器下发的64位前缀，合成一个固定的IPv6地址。

作为服务端，【无状态EUI-64地址】是较为适合的，Linux发行版很多组件（systemd-netword，dhcpcd等）默认都采用EUI-64地址。 

另外还有通过DHCPv6下发的地址，可以通过设置静态分发，对应设备（DUID）下发特定地址。作为服务器地址最适合的方式。

此外，家用宽带ISP提供的IPv6前缀是不定期变化的。可见要访问家庭宽带内网的设备，光是地址就存在了蛮多的变化因素。

### IPv6的【隐私扩展地址】

终端设备，比如手机、工作站版本Windows等设备，则使用【隐私扩展】的方式随机生成64位后缀，这样终端的地址每次链接时候都会随机改变，访问外部资源时候可避免被追踪。 如果要连接Windows远程桌面，安装的是工作站版本，系统默认已经启用【隐私扩展】，主机地址就是随机变化的，想要连接3389就很麻烦了，不过这个特性可以关闭。服务器版本的Windows默认不启用隐私扩展，而家庭版Windows不支持远程桌面\[doge\]。 管理员权限的CMD下执行 
```
netsh interface ipv6 set global randomizeidentifiers=disabled store=active 
netsh interface ipv6 set global randomizeidentifiers=disabled store=persistent 
netsh interface ipv6 set privacy state=disabled store=active 
netsh interface ipv6 set privacy state=disabled store=persistent
```

### IPv6防火墙ip6tables

要从外网通过IPv6访问家里路由器下的设备，最关键一点是路由器上的防火墙要允许这样的转发。 Padavan/Openwrt都是基于Linux - ip6tables的防火墙。

默认情况下，只允许了v6子网内的设备被ping，只允许特定类型的ICMPv6报文通过转发，其他通信报文一律丢弃了。所以虽然IPv6下每个设备都有公网地址，但是还不至于不安全到每个设备都可让人随便连。

### 动态匹配EUI-64后缀

考虑到前缀变化因素，要访问特定设备，就是让IPTABLES匹配特定设备的EUI-64后缀放通这个地址： 
```
ip6tables -I FORWARD -d ::abcd:1234:5678:90ef/::ffff:ffff:ffff:ffff -j ACCEPT
```

可见iptables对v6地址的匹配**掩码**可以非常灵活，不像v4下只按前缀适配。坑的就是这个特征是没有文档的，目前文档中写的mask解释还是适配IPv4的内容，[有人专门发邮件去netfilter列表问了才知道](http://blog.dupondje.be/?p=17)。IPv6地址中，双冒号::的写法代表是前/后均为0位，双冒号只能出现一次。


### 动态匹配DHCPv6固定后缀

对于内网服务器，可以设置DHCPv6进行固定网段后缀分配，比如

```
240e:1234:5678:1234::1024:101
240e:1234:5678:1234::1024:102
240e:1234:5678:1234::1024:103
240e:1234:5678:1234::1024:104
...
```

在路由器上可以一个命令匹配

```
ip6tables -I FORWARD -d ::1024:0000/::ffff:0000 -j ACCEPT
```

### Openwrt中配置转发规则

![](openwrt-ip6tables.png)

### Padavan中设置转发规则

其实padavan中的防火墙功能并没有配置地址匹配转发规则的功能界面，只能在自定义脚本中写原始的iptables命令。截图中使用的padavan是增加了QOS组件的老毛子版本。   

![padavan-ip6tables](padavan-ip6tables.png) 
以上。
