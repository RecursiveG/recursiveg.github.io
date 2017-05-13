---
layout: post
title: 借助IPsec和strongSwan建立隧道并分配IPv6地址
comments: true
categories: Informatics
tags: [IPsec,Networking,教程]
keywords: [IPsec,strongSwan,IPv6,certtool,隧道]
description: 'Setup an IPv6 tunnel through IPsec & strongSwan'
date: 2016-12-24 15:54:08
---
## 准备
在一年前，我写过一篇[文章](/2015/09/get-ipv6-via-gre-tunnel/)，介绍利用GRE隧道将一台服务器的IPv6地址“分配”给另一台电脑，令其能访问IPv6网络的方法。
不过那种方法存在一些问题：

- 不能通过NAT
- 数据不加密
- 需要在服务器手动更新IP

于是热爱折腾~~作死~~的我研究了一下使用IPsec配合IKEv2对流量进行加密的方法。

服务器与本地均为ArchLinux（Arch大法好），strongSwan软件包可从AUR安装。
服务器需要至少有一个公网IPv4和一段Routed IPv6 Subnet。
<!--more-->

## 生成密钥
我们一共需要三对“密钥-证书”对：
- CA密钥和证书：用于签署其它的证书，同时CA证书需要分发到所有机器上。
- 服务器密钥和证书
- 客户端密钥和证书

我使用了ECC证书，因为其具有更短的长度。如果老版本不支持ECC的，也可以使用RSA证书。
先生成三把私钥：

    certtool --generate-privkey --ecc --outfile ca.key
    certtool --generate-privkey --ecc --outfile server.key
    certtool --generate-privkey --ecc --outfile client.key

然后自签名CA证书，`Common Name`可以随意填，但是和之后的配置一定要统一：

    certtool --generate-self-signed --load-privkey ca.key --outfile ca.crt

接着再用CA证书签名其它两把密钥，`Common Name`同样可以随意填，但是不要一样：

    certtool --generate-certificate --load-ca-privkey ca.key --load-ca-certificate ca.crt --load-privkey server.key --outfile server.crt
    certtool --generate-certificate --load-ca-privkey ca.key --load-ca-certificate ca.crt --load-privkey client.key --outfile client.crt

这样就一共产生了六个文件，保存备用。

## 服务端设置
首先需要把密钥文件放到对应的位置：

- `ca.crt`放入`/etc/ipsec.d/cacerts/`
- `server.key`放入`/etc/ipsec.d/private/`
- `server.crt`放入`/etc/ipsec.d/certs/`

然后编辑`/etc/ipsec.secrets`文件，注意空格

    "CN=IPsec server" : ECDSA "server.key"

前面`CN=...`那一串是证书的Subject，CN即Common Name，可以通过`certtool -i < server.crt`查看。

最后编辑`/etc/ipsec.conf`文件：

```
config setup
    charondebug="cfg 2, dmn 2, ike 2, net 2"

conn serverconn #此处是链接名称，可以自由填写
    left=%any
    leftcert=server.crt
    leftid="{这边填入server.crt的Subject}"
    leftca="{这边填入ca.crt的Subject}"
    leftsubnet=::/0     #表示整个IPv6网络都在这端

    right=%any
    rightca="{这边填入ca.crt的Subject}"
    rightsourceip=2001:abc:def:123:456::/80 #客户端IP所在的/80段

    auto=add
    keyexchange=ikev2
    ike=aes256gcm128-sha2_512-modp4096! #选择你喜欢的加密方法
```

然后打开IPv6 Forwarding并启动服务

    sudo sysctl net.ipv6.conf.all.forwarding=1
    sudo systemctl start strongswan

## 客户端设置
步骤基本相同。

- `ca.crt`放入`/etc/ipsec.d/cacerts/`
- `client.key`放入`/etc/ipsec.d/private/`
- `client.crt`放入`/etc/ipsec.d/certs/`
- 编辑`ipsec.secrets`为`"CN=..." : ECDSA "client.key"`

编辑`/etc/ipsec.conf`
```
config setup
    charondebug="cfg 2, dmn 2, ike 2, net 2"

conn clientconn #此处是链接名称，可以自由填写
    left=%any
    leftcert=client.crt
    leftsourceip=%config6 #从服务器获取一个IPv6地址

    right={这里填上你服务器的IPv4地址}
    rightid="{这边填入server.crt的Subject}"
    rightsubnet=::/0 #表示整个IPv6网络都在另一端

    auto=start        #自动连接
    dpdaction=restart #自动重连
    keyexchange=ikev2
    ike=aes256gcm128-sha2_512-modp4096!
```

然后执行`sudo ipsec start --nofork`，如果出现`keeping connection path`字样应该就连接成功了。网卡上会出现一个新的IPv6地址，然后就可以直接访问IPv6网络了。

如果连接不成功或者是无法访问网络，可以考虑检查一下防火墙是不是把数据包drop了。

## References

- [strongSwan Wiki](https://wiki.strongswan.org/projects/strongswan/wiki) strongSwan的官方文档库，同时提供了很多IPsec的资料
- [certtool使用手册](https://www.gnutls.org/manual/html_node/certtool-Invocation.html)
