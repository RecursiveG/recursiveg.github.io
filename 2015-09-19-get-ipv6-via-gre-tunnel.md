---
layout: post
title: Linux下建立GRE隧道并获取IPv6地址
comments: true
categories: Informatics
tags: [Linux,"GRE Tunnel",IPv6]
keywords: ["GRE Tunnel",IPv6,隧道,邻居发现,6to4,6in4]
description: '如何在Linux服务器上建立隧道并获得IPv6地址'
date: 2015-09-19 14:35:10
---
虽然HE有提供免费的Tunnelbroker，不过那速度实在不怎么样。于是考虑在有IPv6地址托管主机上建立一个GRE Tunnel。
GRE Tunnel需要有内核模块`ip_gre`支持。远程主机有一段/64的IPv6，我将其中的一段/80分配给自己的机器。
使用iproute2工具。当然，你自己的机器需要有一个公网IPv4地址。

1. 服务器的公网IPv4是`$server_ipv4`
2. 自己电脑（或者路由器）的公网IPv4是`$client_ipv4`。
3. 服务器的IPv6段是`a:b:c:d::/64`
4. 要分配下去的IPv6段是`a:b:c:d:e::/80`

## 服务器配置
脚本如下，需要root，建议用`sudo -i`：
``` bash
ip tunnel add gre-tunnel mode gre remote $client_ipv4 ttl 64
ip link set gre-tunnel up
ip addr add a:b:c:d:e::1/80 dev gre-tunnel
```

* 第一行建立隧道，`gre-tunnel`是隧道名称，可以按自己喜欢的来，记得其他的也要一起改
* 第二行激活隧道
* 第三行分配IP地址

<!--more-->
## 本地配置
脚本如下，和服务端配置几乎一样，同样需要root：
``` bash
ip tunnel add gre-tunnel mode gre remote $server_ipv4 ttl 64
ip link set gre-tunnel up
ip addr add a:b:c:d:e::2/80 dev gre-tunnel
ip -6 route add default dev gre-tunnel
```

* 第一行建立隧道，隧道名称不必和服务器的一样
* 第二行激活隧道
* 第三行分配IP地址，注意不要和服务器的冲突，这个IP也是将要暴露在网络上的IP
* 第四行设定路由，让IPv6流量都走隧道

## 访问网络
现在，两台机器应该可以互ping了。有的比较奇葩的情况可能需要手动`ip link set gre0 up`一下，gre0似乎是内核模块自动加入的玩意儿，具体怎么回事我也不清楚--\_--|
但是现在还不能访问外网，还需要在服务器执行以下命令：
``` bash
sysctl net.ipv6.conf.all.forwarding=1
sysctl net.ipv6.conf.all.proxy_ndp=1
ip -6 neigh add proxy a:b:c:d:e::2 dev eth0
```
第一行开启forward
二三行和IPv6的NDP(邻居发现)有关，又是个没搞明白的东西真是残念......
eth0是服务器实际连接网络的接口。

## 删除 & 修改
要删除Tunnel，在两端均执行:

    ip link set gre-tunnel down
    ip tunnel del gre-tunnel

如果客户端IP变化:

    ip tunnel change gre-tunnel remote $new_client_ipv4

## 注意事项
虽然叫做“隧道”，但是内容依然是*明文*，对保密要求高的同学们要注意了。
另外直接用命令建立的隧道在重启后会没有，所以可以考虑用networkd之类的东西来管理。
Linux在访问有IPv6地址的域名时会优先使用IPv6，所以要当心服务器流量爆炸。当然配置成IPv4优先也是可以的。
如果你的本地IPv4经常变动的话，你可能需要些脚本之类的东西自动更新服务器的Remote IP。
对于每一个新的IP(新的设备)，都需要在服务端执行`ip -6 neigh add`，有知道怎么解决这个问题的请务必留言...
