---
title: 基于DNSmasq构建本地DNS服务器
date: 2020-07-05 15:22:39
tags:
---

## 本地网络规划方案

为了解决KX上网的问题，经常需要修改DNS设置，干脆就在自己的局域网搭建一个本地DNS服务器，作为今后Linux系统安装的基础设施，一劳永逸解决问题。

原本想用BIND（named），虽然功能强大但是安装配置太复杂，研究发现局域网直接用DNSmasq最合适。

局域网Domain：  caogo.lan
DNS服务器：     192.168.0.130， dnsmasq, dnsmasq.caogo.lan
> 局域网的Domain要明显区别于公网域名，本来考虑`.local`，但是好像 Mac 的 Banjour打印服务使用了这个域名，目前采用`.lan`

## DNSmasq概述

DNSmasq是一个小巧且方便地用于配置DNS和DHCP的工具，适用于小型网络，它提供了DNS功能和可选择的DHCP功能（本案不使用，核心AR路由器负责DHCP）。

它服务那些只在本地适用的域名，这些域名是不会在全球的DNS服务器中出现的。DHCP服务器和DNS服务器结合，并且允许DHCP分配的地址能在DNS中正常解析，而这些DHCP分配的地址和相关命令可以配置到每台主机中，也可以配置到一台核心设备中（比如路由器），DNSmasq支持静态和动态两种DHCP配置方式。

## DNS服务器的配置步骤

1. 设置网卡的静态IP地址130（必须的），并安装必要的基础软件

    ``` sh
    yum install dnsmasq net-tools bind-utils yum-utils tree -y
    ```

2. 编辑DNSmasq核心配置文件`/etc/dnsmasq.conf`

    ``` conf
    conf-dir=/etc/dnsmasq.d,.rpmnew,.rpmsave,.rpmorig
    domain=caogo.lan, 192.168.0.0/24
    expand-hosts
    listen-address=127.0.0.1, 192.168.0.130
    ```

3. 编辑本地域名解析文件`/etc/hosts`

    ``` conf
    127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
    ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

    127.0.0.1       dnsmasq
    192.168.0.130   dnsmasq
    192.168.0.1     gateway
    192.168.0.5     ap
    192.168.0.11    plc

    192.168.0.130   mirror
    192.168.0.130   reg
    192.168.0.130   nfs

    192.168.0.132   master1
    192.168.0.130   worker1
    ```

4. 最关键的一步：编辑域名解析核心配置文件`/etc/resolv.conf`

    ```conf
    domain      caogo.lan

    nameserver  127.0.0.1
    nameserver  8.8.8.8
    nameserver  114.114.114.114
    ```

5. 设置运行环境等

    ``` sh
    # 禁止修改配置文件，以防止NetworkManger等默默修改域名解析配置
    chattr +i /etc/resolv.conf

    # 设置开机自启动
    systemctl enable dnsmasq
    systemctl restart dnsmasq
    ```

6. 确认DNS解析服务运行正常

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
# domain caogo.lan
nameserver 192.169.0.130
```

## 参考资料

- [Dnsmasq的官方网站](https://wiki.archlinux.org/index.php/Dnsmasq#DNS_addresses_file_and_forwarding)
- [Dnsmasq配置的核心参考文档](https://www.howtoing.com/setup-a-dns-dhcp-server-using-dnsmasq-on-centos-rhel)
- [DNSmasq详细解析及详细配置-腾讯云](https://cloud.tencent.com/developer/article/1174717)
- [CentOS 7 安装配置本地DNS (BIND) 服务器（Master-Slave）](https://www.kclouder.cn/centos-7-dns-bind/)
- [BIND的官方网站](https://wiki.archlinux.org/index.php/BIND)
