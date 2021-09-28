---
title: OpenVPN 客户端间选择性互通
date: 2021-09-28 14:33:08
tags: 
    - networking
    - openvpn
    - linux
categories:
    - networking
---

OpenVPN服务器中若配置了`client-to-client`, 终端之间是无限制互通的。
<!-- more -->

配置文档中如此描述：

```
# Uncomment this directive to allow different
# clients to be able to "see" each other.
# By default, clients will only see the server.
# To force clients to only see the server, you
# will also need to appropriately firewall the
# server's TUN/TAP interface.
; client-to-client
```

如果想要指定特定终端能互通，特定不能通，不能打开这个选项。

首先要开启配置`topology subnet`。

此时服务器的tun0上是可以看到终端所有数据包了，即终端数据会进入服务器的内核IP Stack。

```
cat >> /etc/sysctl.conf <<EOF
# for openvpn client packet forward
net.ipv4.ip_forward = 1
EOF
sysctl -p /etc/sysctl.conf
```

最后通过iptables的FORWARD规则，屏蔽特定不允许互通的网段。即可实现。

```
iptables -A FORWARD -i tun0 -o tun0 -s 10.15.100.0/24 -d 10.15.100.0/24 -j DROP
```

原理摘自[Stackoverflow一篇帖子](https://serverfault.com/questions/736274/openvpn-client-to-client)，摘抄如下。

```
If client-to-client is enabled, the VPN server forwards client-to-client packets internally without sending them to the IP layer of the host (i.e. to the kernel). The host networking stack does not see those packets at all.

           .-------------------.
           | IP Layer          |
           '-------------------'


           .-------------------.
           | TUN device (tun0) |
           '-------------------'


           .-------------------.
           | OpenVPN server    |
           '-------------------'
             ^           |
          1  |           |  2   
             |           v
 .----------------.  .----------------.
 | Client a       |  | Client b       |
 '----------------'  '----------------'
If client-to-client is disabled, the packets from a client to another client go through the host IP layer (iptables, routing table, etc.) of the machine hosting the VPN server: if IP forwarding is enabled, the host might forward the packet (using its routing table) again to the TUN interface and the VPN daemon will forward the packet to the correct client inside the tunnel.

           .-------------------.
           | IP Layer          |  (4) routing, firewall, NAT, etc.
           '-------------------'      (iptables, nftables, conntrack, tc, etc.)
              ^          |
          3   |          |  5
              |          v
           .-------------------.
           | TUN device (tun0) |
           '-------------------'
             ^           |
          2  |           |  6  
             |           v
           .-------------------.
           | OpenVPN server    |
           '-------------------'
             ^           |
          1  |           |  7  
             |           v
 .----------------.  .----------------.
 | Client a       |  | Client b       |
 '----------------'  '----------------'
In this case (client-to-client disabled), you can block the client-to-client packets using iptables:

 iptables -A FORWARD -i tun0 -o tun0 -j DROP
where tun0 is your VPN interface.
```