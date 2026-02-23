---
title: 解决 PVE 虚拟机复制时 DHCP 地址重复的问题
date: 2026-02-23 17:36:45
tags:
---

前几天，重新制作了 Debian 12 的基线模版，意外发现 clone 多个虚拟机时，它们的 IP 地址都是相同的，导致无法正常 ssh 登录。
检查 PVE 宿主机的 cloudinit 配置，其 MAC 地址各不相同，IP 都是 DHCP 模式，为什么呢？

`systemd-networkd`是 Debian 12 的默认网络管理软件，根据 [systemd.network官方文档](https://manpages.debian.org/buster/systemd/systemd.network.5.en.html)，DHCP 配置有个`ClientIdentifier`选项，取值范围是[‘mac’, ‘duid’, 'duid-only']。

> The DHCPv4 client identifier to use. Takes one of "mac", "duid" or "duid-only". If set to "mac", the MAC address of the link is used. If set to "duid", an RFC4361-compliant Client ID, which is the combination of IAID and DUID (see below), is used. If set to "duid-only", only DUID is used, this may not be RFC compliant, but some setups may require to use this. Defaults to "duid".

也就是说，如果设置了 ClientIdentifier=duid ，那么当 DHCPv4 客户端在获取动态 IPv4 地址时， 会向 DHCPv4 服务器发送 DUID(DHCP Unique Identifier) 以及网络接口的 IAID(Identity Association Identifier)。 DHCP 服务器可根据 DUID 与 IAID 来唯一定位主机及其网络接口。

实际上，这次出问题的虚拟机模版克隆了多个 VM 之后，在向 DHCP 服务器申请 IP 地址时，其上报的 Client ID 根本不是 VM 的虚拟化 Mac 地址，而是基于 machine-id 生成的、完全相同的 DUID，因此 DHCP 服务器每次都分配完全相同的 IP 地址，这就问题的根本原因。

## 一、systemd 的 machine id

`machine id` 是 Systemd 项目引入的系统标识机制，最早出现在 systemd v197 版本（2012 年），是一个 32 位的 16 进制字符串，用于操作系统安装或启动时的唯一标识。参见 [Debian 关于 machine-id 的技术说明](https://wiki.debian.org/MachineId)。

由于历史原因，`machine-id` 在 Linux 内部有两种实现,分别为 systemd 的 `/etc/machine-id` 和 dbus 的 `/var/lib/dbus/machine-id`，系统命令 `systemd-machine-id-setup` 和 `dbus-uuidgen` 各自尝试复制对方的机器 ID，以避免两个文件顺序相反的进程对 machine id 的判断不一致。

```bash
root@Copy-of-VM-Debian13-Cloud-Base:~# cat /etc/machine-id
d31ba602070b463aac004744a10d3e28

root@Copy-of-VM-Debian13-Cloud-Base:~# ls -l /var/lib/dbus/machine-id
lrwxrwxrwx 1 root root 15 Feb 23 10:25 /var/lib/dbus/machine-id -> /etc/machine-id
```

注意！将 `/etc/machine-id` 设置为 0 字节的空文件，被认为是清除该文件的规范方法，而不是直接删除，因为如果 systemd 运行在完全只读的根文件系统上，它有代码可以在 tmpfs 上创建机器 ID，并将其绑定挂载到空文件之上。

## 二、DHCP 的 DUID 和 IAID

关于 DUID、IAID 的定义，在 [networkd.conf 官方文档](https://manpages.debian.org/buster/systemd/systemd.network.5.en.html)中做了介绍，完全符合 [RFC 4361 : Node-specific Client Identifiers for DHCPv4](https://datatracker.ietf.org/doc/html/rfc4361) 的技术规范。

传统 DHCPv4 用 Client-ID 标识客户身份，格式是 Tpye（0x00）+ 6字节的 MAC 地址；但多网卡、双栈（IPv4+IPv6）、PXE 多阶段启动时，同一设备会出现多个不同 Client-ID，导致 DHCP 服务器分配不同 IP / 租约，造成身份割裂。

RFC 4361 的核心目标是让 DHCPv4 客户端使用 DHCPv6 定义的 DUID 作为唯一标识，为此新定义了 DUID-based Client-ID 替代 Client-ID ，用于标识设备 + 网卡，全局唯一且跨协议一致。

```plain
Client-ID (Option 61) = Type (1字节)  +  IAID (4字节)  +  DUID (变长)
```

- `Type`：固定为 0xFF
- `IAID（Interface Association ID）`：，4 字节，用于区分同一设备的不同网卡（同一设备多网卡用相同 DUID、不同 IAID）。
- `DUID（DHCP Unique ID）`：≥ 6 字节，基于 `RFC 3315` 定义的 DHCPv6 全局唯一标识符，设备生成后永久保存。

![DUID](DUID.png)

对客户端行为规范的核心要求是：

- DHCPv4 客户端：
    必须生成 / 使用 DUID，并持久化存储（重启、重装系统不变）。
    同一设备的所有网卡共用同一个 DUID，但每个网卡用不同 IAID。
    发送 DHCPv4 报文时，Option 61 必须用 255 + IAID + DUID 格式。
    双栈设备（同时跑 DHCPv4/6）必须共用同一个 DUID，无需人工配置。
- DHCPv6 客户端：
    应提供查看 / 配置 DUID 的接口。
    与 DHCPv4 共用 DUID，保证身份统一。
- 阶段网络启动（PXE）
    第一阶段（如 PXE ROM）与后续阶段（如 OS 内 DHCP 客户端）应使用相同 DUID，避免身份切换导致租约冲突。
- DHCP 服务器：
    支持识别 Type=255 的 Client-ID，按 DUID+IAID 做身份绑定，而非仅 MAC。
    对同一 DUID、不同 IAID 的请求，视为同一设备的不同网卡，分配不同 IP 但保持配置一致性。
    兼容传统 Type=0（MAC）的 Client-ID，保证向后兼容。

```console
root@Copy-of-VM-Debian12-cloudinit-v2:/var/lib/cloud/instances# networkctl status eth0 
● 2: eth0                                                                        
                     Link File: /usr/lib/systemd/network/99-default.link
                  Network File: /run/systemd/network/10-netplan-eth0.network
                         State: routable (configured)
                  Online state: online                                           
                          Type: ether
                          Path: pci-0000:00:12.0
                        Driver: virtio_net
                        Vendor: Red Hat, Inc.
                         Model: Virtio network device
             Alternative Names: enp0s18
                                ens18
              Hardware Address: bc:24:11:5c:c5:b7 (Proxmox Server Solutions GmbH)
                           MTU: 1500 (min: 68, max: 65535)
                         QDisc: fq_codel
  IPv6 Address Generation Mode: eui64
      Number of Queues (Tx/Rx): 1/1
              Auto negotiation: no
                       Address: 192.168.31.101 (DHCP4 via 192.168.31.1)
                                fd5c:e694:2545:482d:be24:11ff:fe5c:c5b7
                                fe80::be24:11ff:fe5c:c5b7
                       Gateway: 192.168.31.1
                           DNS: 192.168.31.3
                                8.8.8.8
             Activation Policy: up
           Required For Online: yes
           >>> DHCP4 Client ID: IAID:0xca53095a/DUID <<<
         >>> DHCP6 Client DUID: DUID-EN/Vendor:0000ab11 d0b12f38460139d9 <<<
```

***一句话总结***：RFC 4361 是 DHCPv4 的身份标准化补丁：用 DHCPv6 的 DUID 做全局唯一 ID，搭配 IAID 区分网卡，让设备在 IPv4/IPv6、多网卡、多启动阶段都保持同一个网络身份，解决租约与配置的一致性难题。

### 三、解决方案

解决办法是修改 Debian 12 的基线模版，在最后清理阶段增加如下步骤：

1. 清空 machine-id 配置：后续克隆后的 VM 启动时，系统将自动生成全新的、不一样的 machine-id，DHCP 服务器就可以区分不同 VM 了
2. 清空 cloud-init 日志：这是保险起见，避免遗留的日志信息干扰 VM 启动
3. 基线模版直接关机！！！并转为模版避免修改

```bash
rm -f /etc/machine-id /var/lib/dbus/machine-id
touch /etc/machine-id

cloud-init clean --logs
rm -rf /var/lib/cloud/*
```

---

## 附录：DHCP 服务响应信息查询

nmap（Network Mapper）是 Debian 官方 main 仓库中的网络扫描与审计工具包，核心功能是：

- 端口扫描（TCP/UDP）、服务版本探测、操作系统指纹识别；
- 网络拓扑发现、存活主机检测、漏洞扫描（需配合 NSE 脚本）；
- 支持 IPv4/IPv6，兼容各类网络协议（DHCP、DNS、HTTP 等）。

检查 DHCP 服务器相应信息，可以使用命令：```nmap --script broadcast-dhcp-discover```
这个会直接列出所有给你分配地址的 DHCP 服务器 IP，如果出现多个 server-identifier 就是多 DHCP 冲突。

```console
root@Copy-of-VM-Debian12-cloudinit-v2:/var/lib/systemd# nmap --script broadcast-dhcp-discover
Starting Nmap 7.93 ( https://nmap.org ) at 2026-02-23 23:33 CST
Pre-scan script results:
| broadcast-dhcp-discover: 
|   Response 1 of 1: 
|     Interface: eth0
|     IP Offered: 192.168.31.97                             # 返回的可分配 IP 
|     DHCP Message Type: DHCPOFFER
|     Server Identifier: 192.168.31.1                       # DHCP Server
|     IP Address Lease Time: 12h00m00s
|     Renewal Time Value: 6h00m00s
|     Rebinding Time Value: 10h30m00s
|     Subnet Mask: 255.255.255.0
|     Broadcast Address: 192.168.31.255
|     Vendor Specific Information: miwifi-RC01-1.1.56-100
|     Hostname: MiWiFi-RC01-srv
|     Domain Name Server: 192.168.31.3, 8.8.8.8             # 返回的默认 DNS Server
|_    Router: 192.168.31.1                                  # 返回的默认 Gateway
WARNING: No targets were specified, so 0 hosts scanned.
Nmap done: 0 IP addresses (0 hosts up) scanned in 10.49 seconds
```

---

## 参考文献

- [解决方案: 虚拟机批量 clone 导致 machine-id 引起的 ip 异常 问题](https://kms.app/archives/456/)
- [DUID的算法分析](https://unix.stackexchange.com/questions/743776/systemd-network-duid-iaid-and-dhcpv4-clientidentifier)
