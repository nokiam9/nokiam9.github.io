---
title: 基于DNSmasq构建本地DNS服务器
date: 2020-07-05 15:22:39
tags:
---

## 本地网络规划方案

为了解决KX上网的问题，经常需要修改DNS设置，干脆就在自己的局域网搭建一个本地DNS服务器，作为今后Linux系统安装的基础设施，一劳永逸解决问题。

局域网Domain：  caogo.lan
DNS服务器：     192.168.0.144， dnsmasq, dnsmasq.caogo.lan

原本想用BIND（named），虽然功能强大但是安装配置太复杂，研究发现局域网直接用DNSmasq最合适。
另外，局域网的Domain本来考虑`.local`，但发现Mac的Banjour服务默认使用了该域名后缀

> Bonjour服务是基于mDNS(Multicast DNS)协议实现的，mDNS协议适用于局域网内的设备通过组播的方式交互DNS记录来完成域名解析，约定的组播地址是：224.0.0.251，端口号是5353，mdns协议使用DNS协议一样的报文格式。详细资料见参考目录

## DNSmasq概述

DNSmasq是一个小巧且方便地用于配置DNS和DHCP的工具，适用于小型网络，它提供了DNS功能和可选择的DHCP功能（本案不使用，核心AR路由器负责DHCP）。

它服务那些只在本地适用的域名，这些域名是不会在全球的DNS服务器中出现的。DHCP服务器和DNS服务器结合，并且允许DHCP分配的地址能在DNS中正常解析，而这些DHCP分配的地址和相关命令可以配置到每台主机中，也可以配置到一台核心设备中（比如路由器），DNSmasq支持静态和动态两种DHCP配置方式。

## DNS服务器的配置步骤

1. 安装操作系统后，找到网卡参数文件并设置静态IP地址130（必须的），至少包含以下参数：BOOTPROTO、ONBOOT、IPADDR、NETMASK、GATEWAY。
   然后，激活该网络配置`systemctl restart network`并确认成功联网。

2. 刚装好的Centos必须马上关闭firewalld，否则后面虽然netestat显示53端口开放，但是外网死活就访问不了
   为了后续yum安装访问Internet，临时设置域名服务器安装必要的基础网络工具。

    ``` sh
    # 赶紧关闭防火墙
    sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
    setenforce 0
    systemctl disable firewalld
    systemctl stop firewalld

    # 为安装工具软件，临时设置DNS
    cat > /etc/resolv.conf << EOF
    nameserver 8.8.8.8
    EOF

    yum install dnsmasq net-tools bind-utils yum-utils tree -y
    ```

3. 定义DNSmasq核心配置文件`/etc/dnsmasq.conf`

    ``` sh
    mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig

    cat > /etc/dnsmasq.conf << EOF
    listen-address=127.0.0.1, 192.168.0.144
    expand-hosts
    domain=caogo.local
    server=8.8.8.8
    server=114.114.114.114
    address=/caogo.local/127.0.0.1
    address=/caogo.local/192.168.0.144
    EOF
    ```

4. 导入本地域名解析文件`/etc/hosts`，先定义这些hosts，以后要增加就编辑这个文件

    ``` sh
    cat >> /etc/hosts << EOF

    127.0.0.1       dnsmasq
    192.168.0.1     gateway
    192.168.0.5     ap
    192.168.0.8     proxy
    192.168.0.11    plc

    192.168.0.132   pve01 pve

    192.168.0.120   nfs
    192.168.0.140   ntp
    192.168.0.144   dnsmasq dns ns01 ns02
    192.168.0.148   mirror yum

    192.168.0.210   master1
    192.168.0.212   worker1
    EOF
    ```

    > 注意：在PVE虚拟机中，Cloud-init启动会修改`/etc/hosts`，解决办法是修改 `/etc/cloud/cloud.cfg`，注释其中的`update_etc_hosts`字段

5. 最关键的一步：编辑域名解析核心配置文件`/etc/resolv.conf`

    ``` sh
    cat > /etc/resolv.conf << EOF
    # set localhost as the unique namserver
    nameserver  127.0.0.1
    EOF
    ```

6. 设置运行环境等

    ``` sh
    # 禁止修改配置文件，以防止NetworkManger等默默修改域名解析配置
    chattr +i /etc/resolv.conf

    # 设置开机自启动
    systemctl enable dnsmasq
    systemctl restart dnsmasq
    ```

7. 确认DNS解析服务运行正常

    ``` sh
    [root@localhost ~]# netstat -tunpl |grep 53
    tcp        0      0 0.0.0.0:53              0.0.0.0:*               LISTEN      9863/dnsmasq
    tcp6       0      0 :::53                   :::*                    LISTEN      9863/dnsmasq
    udp        0      0 0.0.0.0:53              0.0.0.0:*                           9863/dnsmasq
    udp6       0      0 :::53                   :::*                                9863/dnsmasq  
    ```

## 如何使用DNS公共服务

在局域网中，为每台服务器的`/etc/resolv.conf`中设置：

``` conf
# domain caogo.local
nameserver 192.168.0.144
```

> 在创建虚拟机时，cloudinit 将通过 DHCP 获取网络配置，从而修改 /etc/resolv.conf 文件，为此建议模版中将该文件设置为不可修改，方法是 `chattr +i /etc/resolv.conf`

## 参考资料

- [Dnsmasq的官方网站](https://wiki.archlinux.org/index.php/Dnsmasq#DNS_addresses_file_and_forwarding)
- [Dnsmasq配置的核心参考文档](https://www.howtoing.com/setup-a-dns-dhcp-server-using-dnsmasq-on-centos-rhel)
- [DNSmasq详细解析及详细配置-腾讯云](https://cloud.tencent.com/developer/article/1174717)
- [CentOS 7 安装配置本地DNS (BIND) 服务器（Master-Slave）](https://www.kclouder.cn/centos-7-dns-bind/)
- [BIND的官方网站](https://wiki.archlinux.org/index.php/BIND)
- [局域网设备发现之Bonjour协议](https://blog.csdn.net/yueqian_scut/article/details/52694411)
- [RFC-6762](https://en.wikipedia.org/wiki/Multicast_DNS)
