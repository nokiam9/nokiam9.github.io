---
title: OpenVPN + OpenWrt 的内网穿透方案
date: 2025-01-22 20:11:04
tags:
---

## 二、Home Client 安装

### 路由器 IP 地址配置

```bash
uci set network.lan.proto='static'
uci set network.lan.ipaddr='192.168.73.3'
uci set network.lan.netmask='255.255.255.0'
uci set network.lan.gateway='192.168.73.1'
uci set network.lan.dns='192.168.73.1'
uci commit network
service network restart
```

### 路由器 Firewall 放通

```txt
config zone
    option name 'wan'
    option input 'REJECT'
    option output 'ACCEPT'
    option forward 'REJECT'
    option masq '1'
    option mtu_fix '1'
    list device 'tun0'

config forwarding
    option src 'lan'
    option dest 'wan'

config forwarding
    option src 'wan'
    option dest 'lan'
```

### MacOS 清理 DNS 缓存的命令

``` sh
sudo killall -HUP mDNSResponder
```

[OpenWrt：关于 OpenVPN 扩展的说明](https://openwrt.org/docs/guide-user/services/vpn/openvpn/extras)