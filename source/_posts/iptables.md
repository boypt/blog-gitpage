---
title: iptables
date: 2021-09-07 10:39:44
tags: 
	- iptables
	- linux
---

# Packet Flow Chart

![](packet_flow10.png)

# Netfilter Flow Chart

![](Netfilter-packet-flow.svg.png)

# Firewall Rules

##
```
apt install iptables-persistent

```

#  NAT as port forwarding 
```
# 数据包进入PREROUTING Chain，DNAT修改来源数据包的目的地址/端口为映射的$DEST_IP:$PORT
iptables -t nat -A PREROUTING -p tcp --dst $WAN_IP --dport 80 -j DNAT --to-destination $DEST_IP:$PORT

# 此时Packet的目的地址不是本机地址，而是$DEST_IP，进入filter表的FORWARD Chain进行规则审核，要允许其通过（若filter表已是默认允许的，可以忽略本条）
iptables -A FORWARD -p tcp --dst $DEST_IP --dport $PORT -j ACCEPT

# 进入POSTROUTING Chain，SNAT修改数据包中的来源地址为本网关；若目的机的默认网关就是本机，可以忽略本步（因为如果是目的机的默认网关，不管发往哪里的包都是发回来本网关；不然的话会发去了另外一个网关，无法成为相同一个NAT会话，无法通信）。默认网关的方式不用这句时目标机可以看到来源的真实地址。
iptables -t nat -A POSTROUTING -p tcp --dst $DEST_IP --dport 80 -j SNAT --to-source $LAN_IP

# 在网关本机和内网其他机器访问WAN_IP这个端口映射，数据包产生在OUTPUT Chain，需要做和PREROUTING相同的操作才能访问到（若不需要，可忽略本步）
iptables -t nat -A POSTROUTING --dst $WAN_IP -p tcp --dport 80 -j DNAT --to-destination $DEST_IP:$PORT
```


# NAT as gateway 

Enable IP Forwarding
```
sed -i 's/.*net\.ipv4\.ip_forward.*/net.ipv4.ip_forward = 1/' /etc/sysctl.conf
sysctl -p
```

MASQUERADE, pppoe等动态IP环境使用环境

```
iptables -t nat -A POSTROUTING -s 10.0.0.0/8 -j MASQUERADE
```

or

SNAT, 静态外网IP, 或出口网卡绑定了多个IP时候使用
```
iptables -t nat -A POSTROUTING -s 10.0.0.0/8 -j SNAT --to-source <IP>
```

自动调整经pppoe-wan接口发出的TCP数据MSS, PPP链路情况
```
iptables -t mangle -I POSTROUTING -o pppoe-wan -p tcp -m tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```


#  NAT as proxy （双向NAT，举例推特API服务器hosts转发） 
```
iptables -t nat -A PREROUTING -d [YOUR SERVER IP] -p tcp -m tcp --dport 443 -j DNAT --to-destination 199.59.148.20:443 
iptables -t nat -A POSTROUTING -d 199.59.148.20 -p tcp -m tcp --dport 443 -j SNAT --to-source [YOUR SERVER IP]

iptables -A FORWARD -d 199.59.148.20 -p tcp -m tcp --dport 443 -j ACCEPT 
iptables -A FORWARD -s 199.59.148.20 -p tcp -m tcp --sport 443 -m state --state RELATED,ESTABLISHED -j ACCEPT 

```

#  Iptables Firewall 后 FTP 服务 List 命令超时 

```
modprobe ip_conntrack_ftp
echo "ip_conntrack_ftp" >>/etc/modules
```

# Filter DNS from GFW
```
iptables -A INPUT --source 8.8.8.8,8.8.4.4 -p udp --source-port 53 -m dscp ! --dscp 0x00 -j DROP
iptables -A INPUT --source 8.8.8.8,8.8.4.4 -p udp --source-port 53 -m ttl --ttl-gt 48 -j DROP 
```

#  Quick fix for deprecated state module [since Dec 2012]
```
sed -i "s/-m state --state/-m conntrack --ctstate/g" /etc/iptables/iptables.rules
```
