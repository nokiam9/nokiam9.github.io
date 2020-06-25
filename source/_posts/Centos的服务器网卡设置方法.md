---
title: Centos的服务器网卡设置方法
date: 2020-06-25 23:44:17
tags:
---

## 重装Centos系统

使用Centos安装U盘，选择最小化安装，设置root口令，设置失去，最后重新启动。

## 配置有线网卡

安装完成后，编辑网卡配置文件`/etc/sysconfig/network-scripts/ifcfg-eno1`

``` shell
[root@localhost network-scripts]# more /etc/sysconfig/network-scripts/ifcfg-eno1
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static            # 修改默认值dhcp
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=eno1
UUID=47adb195-54ee-4c74-85a2-2b1fcb499e01
DEVICE=eno1
ONBOOT=yes                  # 修改默认值no
IPADDR=192.168.0.132        # New
NETMASK=255.255.255.0       # New
GATEWAY=192.168.0.1         # New
DNS1=192.168.0.1            # New
```
