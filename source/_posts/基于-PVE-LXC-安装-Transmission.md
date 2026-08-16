---
title: 基于 PVE LXC 安装 Transmission
date: 2026-08-16 13:16:41
tags:
---

换了 Debian，安装 Transmission 还挺麻烦，用户名、安装目录、默认配置都与 Centos 差异很大。
尝试使用 PVE 的 LXC，与 VM 相比更轻量化，操作系统也换成更轻量化的 `Alpine 3.31` 来安装 Transmission。

当然，此次安装也发现 PVE LXC 存在很多技术槽点，例如：默认的非特权模式不支持 NFS，需要设置嵌套的虚拟化 nesting，默认提供 SSH 服务等，还偶尔发生系统崩溃的现象。

## 一、transmission 安装

```bash
apk add transmission-daemon
rc-service transmission-daemon start
```

配置文件位于：`/var/lib/transmission/config/settings.json`
下载目录位于：`/var/lib/transmission/downloads/`

根据需要修改配置，但注意**必须停止服务**，否则重启服务后修改又被默认配置覆盖！

```yaml
"rpc-host-whitelist-enabled": false,        # 是否只允许白名单的主机名称，即反向代理需要核对域名
"rpc-whitelist-enabled": false,             # 是否只允许白名单（默认本机 127.0.0.1）地址接入
"rpc-authentication-required": false,       # 是否需要用户鉴权，用户名和密码在配置文件中，且每次启动改变
```

重启服务生效配置，并设置自动启动

```bash
rc-service transmission-daemon start
rc-update add transmission-daemon
```

## 二、Samba 安装

```bash
apk add smaba4
rc-service samba start
```

配置文件位于：`/etc/samba/smb.conf`\
新增 BT 下载目录的网络共享：

```yaml
[transmission]                                               
    comment = transmission
    path = /var/lib/transmission/downloads/                  
    public = yes                                                  
    browseable = yes                              
    writable = yes
    guest ok = yes                                         
    create mask = 0644                                        
    directory mask = 0775  
```

为便于文件操作，还要配置 [global] 段落设置**匿名用户**访问权限：

```yaml
;   passdb backend =tdbsam       
map to guest = bad user
guest account = transmission  
```

重启服务生效配置，并设置自动启动

```bash
rc-service samba restart
rc-update add samba
```
