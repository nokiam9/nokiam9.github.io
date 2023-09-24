---
title: Centos网络管理要点
date: 2023-08-19 10:58:31
tags:
---

## 一、 网络接口设备

Linux2.6 内核引入了一个基于内存的文件系统 **sysfs** ，用于向用户空间导出内核设备资源并且进行读写操作，挂载点位于`/sys` 。

```console
[root@MiWiFi-RA70-srv ~]# tree /sys -L 1
/sys
├── block           已废弃。记录当前被发现的所有块设备，已迁移到 /sys/class/block
├── bus             已注册在操作系统内核的总线类型，每种类型均包括 devices 和 drivers
├── class           最核心！已注册在操作系统内核的全量设备，按照设备功能分目录管理
├── dev             维护一个字符设备和块设备的软链接文件
├── devices         最基础！全局设备结构体系，包含所有被发现的注册在各种总线上的各种真实物理设备
├── firmware        系统加载固件对象和属性的用户空间接口
├── fs              记录当前被发现的所有文件系统
├── hypervisor      与虚拟化Xen相关的装置
├── kernel          内核中所有可调整的参数
├── module          包含所有的被载入操作系统内核的模块
└── power           系统的电源选项
```

该文件系统是内核设备树的一个直观反映，也就是说，当一个内核对象被创建的时候，对应的文件和目录也在内核对象子系统中被创建。
Centos网络接口信息位于`/sys/class/net`，但该目录下实际都是软链接文件，物理文件实际位于`/sys/devices`。

```console
[root@MiWiFi-RA70-srv fs]# ls -l /sys/class/net
总用量 0
lrwxrwxrwx 1 root root 0  8月 19 10:24 ens18 -> ../../devices/pci0000:00/0000:00:12.0/virtio3/net/ens18
lrwxrwxrwx 1 root root 0  8月 19 10:24 lo -> ../../devices/virtual/net/lo
```

要查看网络接口信息，最简单的方法就是命令`ip a`，或者传统方式`ifconfig -a`

```console
[root@MiWiFi-RA70-srv network]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 5a:3b:ab:a3:01:88 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.32/24 brd 192.168.0.255 scope global dynamic ens18
       valid_lft 28934sec preferred_lft 28934sec
    inet6 2409:8a00:321c:d121:583b:abff:fea3:188/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 215859sec preferred_lft 129459sec
    inet6 fe80::583b:abff:fea3:188/64 scope link 
       valid_lft forever preferred_lft forever
```

## 二、 网络服务组件

### 1. network.service

#### network的启动方式

Centos7 之前的版本都是通过 `network.service` 管理网络配置，启动方式还是基于`rc.d`的传统方式，启动脚本位于 `/etc/rc.d/init.d/network`。

```console
[root@copy-of-vm-centos7 ~]# systemctl status network
● network.service - LSB: Bring up/down networking
   Loaded: loaded (/etc/rc.d/init.d/network; bad; vendor preset: disabled)
   Active: active (running) since 四 2023-08-17 23:53:18 CST; 1 day 16h ago
     Docs: man:systemd-sysv-generator(8)
  Process: 576 ExecStart=/etc/rc.d/init.d/network start (code=exited, status=0/SUCCESS)
   CGroup: /system.slice/network.service
           └─760 /sbin/dhclient -1 -q -lf /var/lib/dhclient/dhclient--eth0.lease -pf /var/run/dhclient-eth0.pid -H copy-of-vm-centos7 eth0

8月 18 22:32:37 copy-of-vm-centos7.8-cloudinit dhclient[760]: bound to 192.168.0.17 -- renewal in 15251 seconds.
8月 19 02:46:48 copy-of-vm-centos7.8-cloudinit dhclient[760]: DHCPREQUEST on eth0 to 192.168.0.1 port 67 (xid=0x1b984451)
8月 19 02:46:48 copy-of-vm-centos7.8-cloudinit dhclient[760]: DHCPACK from 192.168.0.1 (xid=0x1b984451)
8月 19 02:46:50 copy-of-vm-centos7.8-cloudinit dhclient[760]: bound to 192.168.0.17 -- renewal in 15611 seconds.
8月 19 07:07:01 copy-of-vm-centos7.8-cloudinit dhclient[760]: DHCPREQUEST on eth0 to 192.168.0.1 port 67 (xid=0x1b984451)
8月 19 07:07:01 copy-of-vm-centos7.8-cloudinit dhclient[760]: DHCPACK from 192.168.0.1 (xid=0x1b984451)
8月 19 07:07:03 copy-of-vm-centos7.8-cloudinit dhclient[760]: bound to 192.168.0.17 -- renewal in 17059 seconds.
8月 19 11:51:22 copy-of-vm-centos7.8-cloudinit dhclient[760]: DHCPREQUEST on eth0 to 192.168.0.1 port 67 (xid=0x1b984451)
8月 19 11:51:22 copy-of-vm-centos7.8-cloudinit dhclient[760]: DHCPACK from 192.168.0.1 (xid=0x1b984451)
8月 19 11:51:24 copy-of-vm-centos7.8-cloudinit dhclient[760]: bound to 192.168.0.17 -- renewal in 17493 seconds.
```

#### network的配置文件

网卡配置文件位于`/etc/sysconfig/network-scripts/`，每个设备都需要生成独立的接口配置文件，命名方式是`ifcfg-<网络接口名>`。

```console
[root@MiWiFi-RA70-srv /]# ls -l /etc/sysconfig/network-scripts/
总用量 4
-rw-r--r--. 1 root root 279  8月 15 22:11 ifcfg-ens18

[root@MiWiFi-RA70-srv /]# more /etc/sysconfig/network-scripts/ifcfg-ens18
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=dhcp
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens18
UUID=8e4c648c-871f-4223-8d94-ee09aa8580b9
DEVICE=ens18
ONBOOT=no
```

#### network的应用说明

- `network.service` 是一个非常简单的软件包，仅有264行代码的脚本文件
- 设计目标基于简单的有线网络环境，当网络接口配置信息修改后，网络服务必须重新启动，因此很难满足无线网络切换的需求

### 2. NetworkManager.service

RedHat 在2004年启动了 NetworkManager 项目，皆在能够让Linux用户更轻松的处理现代网络需求，尤其是无线网络，能够自动发现网卡并配置IP地址，现在由 GNOME 管理。

#### NetworkManager的启动方式

`NetworkManager.service`的主程序是`/usr/sbin/NetworkManager`，支持 systemd 启动方式，启动配置文件位于`/usr/lib/systemd/system/NetworkManager.service`。

```console
[root@MiWiFi-RA70-srv ~]# systemctl status NetworkManager
● NetworkManager.service - Network Manager
   Loaded: loaded (/usr/lib/systemd/system/NetworkManager.service; enabled; vendor preset: enabled)
  Drop-In: /usr/lib/systemd/system/NetworkManager.service.d
           └─NetworkManager-ovs.conf
   Active: active (running) since Fri 2023-08-18 00:06:55 CST; 33min ago
     Docs: man:NetworkManager(8)
 Main PID: 547 (NetworkManager)
    Tasks: 5
   Memory: 13.0M
   CGroup: /system.slice/NetworkManager.service
           ├─547 /usr/sbin/NetworkManager --no-daemon
           ├─573 /sbin/dhclient -d -q -sf /usr/libexec/nm-dhcp-helper -pf /var/run/NetworkManager/dhclient-eth0.pid -lf /var/lib/NetworkMan>
           └─965 /sbin/dhclient -d -q -6 -N -sf /usr/libexec/nm-dhcp-helper -pf /var/run/NetworkManager/dhclient6-eth0.pid -lf /var/lib/Net>

8月 18 00:07:00 MiWiFi-RA70-srv NetworkManager[547]: <info>  [1692288420.1369] dhcp6 (eth0): option life_starts          => '1692288420'
8月 18 00:07:00 MiWiFi-RA70-srv NetworkManager[547]: <info>  [1692288420.1369] dhcp6 (eth0): option max_life             => '43200'
8月 18 00:07:00 MiWiFi-RA70-srv NetworkManager[547]: <info>  [1692288420.1369] dhcp6 (eth0): option preferred_life       => '43200'
8月 18 00:07:00 MiWiFi-RA70-srv NetworkManager[547]: <info>  [1692288420.1369] dhcp6 (eth0): option rebind               => '34560'
8月 18 00:07:00 MiWiFi-RA70-srv NetworkManager[547]: <info>  [1692288420.1369] dhcp6 (eth0): option renew                => '21600'
8月 18 00:07:00 MiWiFi-RA70-srv NetworkManager[547]: <info>  [1692288420.1369] dhcp6 (eth0): option requested_dhcp6_client_id => '1'
8月 18 00:07:00 MiWiFi-RA70-srv NetworkManager[547]: <info>  [1692288420.1369] dhcp6 (eth0): option requested_dhcp6_domain_search => '1'
8月 18 00:07:00 MiWiFi-RA70-srv NetworkManager[547]: <info>  [1692288420.1370] dhcp6 (eth0): option requested_dhcp6_name_servers => '1'
8月 18 00:07:00 MiWiFi-RA70-srv NetworkManager[547]: <info>  [1692288420.1370] dhcp6 (eth0): option starts               => '1692288420'
8月 18 00:07:00 MiWiFi-RA70-srv NetworkManager[547]: <info>  [1692288420.1370] dhcp6 (eth0): state changed unknown -> bound, event ID="28:3>
[root@MiWiFi-RA70-srv ~]#
```

#### NetworkManager的配置文件

`NetworkManager.service`网卡配置方式与`network.service`保持一致，也是位于`/etc/sysconfig/network-scripts/`，文件命名和配置方式也完全相同。
`nmcli`是用于控制 NetworkManager 和报告网络状态的命令行工具，用于创建，显示，编辑，删除，激活和停用网络连接，以及控制和显示网络设备状态。

```console
[root@MiWiFi-RA70-srv ~]# nmcli connection show
NAME   UUID                                  TYPE      DEVICE 
ens18  8e4c648c-871f-4223-8d94-ee09aa8580b9  ethernet  ens18  

[root@MiWiFi-RA70-srv ~]# ls -l /etc/sysconfig/network-scripts
总用量 4
-rw-r--r-- 1 root root 280  8月 19 23:40 ifcfg-ens18
```

此外，NetworkManager 还有一个漂亮的图形界面软件 `nmtui`，其核心功能与`nmcli`完全相同。
![nmtui](nmtui.png)

#### NetworkManager的应用说明

- Centos7 同时支持`network.service`和`NetworkManager.service`，相当于在 Centos7 的一个过渡，默认情况下这2个服务都有开启，但是因为`NetworkManager.service`当时的兼容性不好，大部分人都会将其关闭
- Centos 8 已经废弃 network.service（默认不安装），只能通过 NetworkManager 进行网络配置，OpenEuler 21.10 也是这样

### 3. systemd-networkd.service

作为一个 “从未完成、从未完善、但一直追随技术进步” 的系统，systemd 已经不只是一个初始化进程，它被设计为一个更广泛的系统以及服务管理平台，包含了不断增长的核心系统进程、库和工具的生态系统。
`systemd-networkd`是 Systemd 计划的一部分，旨在提供一个轻量级、高性能的网络管理器。它使用原生Linux网络接口，不依赖第三方软件包，因此占用更少的资源和内存。

#### systemd-networkd的启动方式

systemd 210 及其更高版本提供了`systemd-networkd`，其主程序是`/usr/lib/systemd/systemd-networkd`，启动配置文件位于`/usr/lib/systemd/system/systemd-networkd.service`。

```console
[root@MiWiFi-RA70-srv ~]# systemctl status systemd-networkd
● systemd-networkd.service - Network Service
   Loaded: loaded (/usr/lib/systemd/system/systemd-networkd.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2023-08-18 00:06:55 CST; 35min ago
     Docs: man:systemd-networkd.service(8)
 Main PID: 548 (systemd-network)
   Status: "Processing requests..."
    Tasks: 1
   Memory: 1.6M
   CGroup: /system.slice/systemd-networkd.service
           └─548 /usr/lib/systemd/systemd-networkd

8月 18 00:06:55 MiWiFi-RA70-srv systemd[1]: Starting Network Service...
8月 18 00:06:55 MiWiFi-RA70-srv systemd-networkd[548]: Enumeration completed
8月 18 00:06:55 MiWiFi-RA70-srv systemd[1]: Started Network Service.
8月 18 00:06:56 MiWiFi-RA70-srv systemd-networkd[548]: eth0: Gained carrier
8月 18 00:06:58 MiWiFi-RA70-srv systemd-networkd[548]: eth0: Gained IPv6LL
```

#### systemd-networkd的配置方式

网络配置文件保存在`/etc/systemd/network/`。
配置文件名可以任意，物理网络设备一般使用后缀名`.network`，虚拟网络设备一般使用后缀名`.netdev`。

```console
$ cat /etc/systemd/network/20-dhcp.network
[Match]
Name=enp3*
[Network]
DHCP=yes

$ cat /etc/systemd/network/10-static-enp3s0.network
[Match]
Name=enp3s0
[Network]
Address=192.168.10.50/24
Gateway=192.168.10.1
DNS=8.8.8.8
```

正如你上面看到的，每个网络配置文件包括了一个或多个 “sections”，每个 “section”都用 [XXX] 开头。每个 section 包括了一个或多个键值对。
`[Match]`部分决定这个配置文件配置哪个（些）网络设备。例如，这个文件匹配所有名称以 ens3 开头的网络设备（例如 enp3s0、 enp3s1、 enp3s2 等等）对于匹配的接口，`[Network]`部分指定的 DHCP 网络配置。

> 配置目录存在多个文件时，`systemd-networkd`会按照字母顺序一个个加载并处理。

#### systemd-networkd的应用说明

- 目前，`systemd-networkd`还不能直接支持无线网络，但你可以使用`wpa_supplicant`服务配置无线适配器，然后把它和`systemd-networkd`联系起来
- 使用`systemd-networkd`时，需要手工启用`systemd-resolved`服务用于域名解析。
    该服务实现了一个缓存式 DNS 服务器，并通过`/run/systemd/resolve/resolv.conf`管理DNS服务。
    由于很多应用程序依赖于`/etc/resolv.conf`，因此为了兼容性需要为两者建立一个软链接。
- Ubuntu 18.04及更高版本中，默认使用 systemd-networkd 作为网络管理器，但也可以选择切换到 NetworkManager。

### 4. netplan

Netplan 是由 Ubuntu 的母公司 Canonical 开发的网络组件，目标是为不同的后端管理工具提供统一的网络配置抽象方法。

- 目前支持的网络管理工具后端为：NetworkManager 和 NetworkManager
- 网络配置语言采用 YAML 格式
- 网络配置存储目录位于：`/etc/netplan/*.yaml`，可以系统管理员手工配置，也可以是云镜像或者其他操作系统部署设施自动生成
- Github 主页位于：[https://github.com/canonical/netplan](https://github.com/canonical/netplan)

![Alt text](netplan_design_overview.svg)

以最简单的静态网卡配置为例，

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:
      addresses:
        - 10.10.10.2/24
      nameservers:
        search: [mydomain, otherdomain]
        addresses: [10.10.10.1, 1.1.1.1]
      routes: 
        - to: default
          via: 10.10.10.1
```

netplan 提供两个命令行工具：

- `netplan generate` ：以 `/etc/netplan` 配置为管理工具生成配置信息
- `netplan apply` ：应用配置。调整 /etc/netplan 配置后，需要执行该命令方能生效，必要时重启管理工具

## 三、小结

1. 如果你使用的是 KDE 或者 GOME 等 Linux 桌面，毫无疑问应该使用 NetworkManager。
    因为其使用DBus作为通信机制，可以集成各种网络类型和设备，包括以太网、无线网络、蓝牙、移动宽带等，支持广泛的网络协议和语言环境，提供图形界面和命令行界面等多种管理方式，能让你的计算机在任何时候通过多种接口保持连接。
2. 如果你使用的是 Linux 服务器，则需要区分不同版本：
    - Centos7 及以前版本：建议直接使用 network，并关闭 NetworkManager，因为存在一些兼容性问题
    - Centos8 及以后版本：建议使用 NetworkManager，并关闭 systemd-networkd（network 已被默认卸载）
3. systemd-networkd 有着快速启动、低内存占用和高性能等突出优点，与其他 systemd 组件（例如用于域名解析的 resolved、NTP 的timesyncd，用于命名的 udevd）结合的非常好，但目前还不是一个成熟的解决方案，主要问题是：
    - 不支持无线网络，需要手工关联`wpa_supplicant`服务
    - 全局域名解析存在缺陷，需要手工关联`systemd-resolved`服务
    - 不能为更高层面的脚本编程提供 ifup/ifdown 钩子函数
4. Netplan 目前并未流行，主要是 Ubuntu 17 以后的版本提供，Centos 似乎没有默认安装包。

---

## 附录：网卡设备的命名方式

Centos6 及之前的版本网卡命名格式：eth[0123…]，也就是常见的`eth0`等等。随着服务器网卡的数量、类型和集成方式越来越复杂（既有主板集成的，也有PCIe插槽，wifi、蓝牙等无线网卡），简单的数字编号已经很难准确标识网卡接口的物理位置。

Centos7 开始默认启用一致网络设备命名（Consistent Network Device Naming），支持 biosdevname 和 net.ifnames 两种命名规范。
常见的 net.ifnames 命名规范表述为：`设备类型 + 设备位置 + 数字`，主要设备类型前缀包括：

- `en` ：用于以太网
- `wl` ：用于无线 LAN(WLAN)
- `ww` ：用于无线广域网(WWAN)。

另外，以下之一会根据 udev 设备管理器应用的模式，附加到上述前缀中的一个：

- `o<on-board_index_number>`：主板集成设备
- `s<hot_plug_slot_index_number>[f<function>][d<device_id>]`：主板集成的PCI插槽
    请注意，所有多功能 PCI 设备在设备名称中都包含`[f<function>]`号，包括功能 0 设备。
- `x<MAC_address>`：网卡的MAC地址
- `[P<domain_number>]p<bus>s<slot>[f<function>][d<device_id>]`：PCI扩展卡
    `[P<domain_number>]` 部分定义了 PCI 的地理位置。如果域号不是 0 ，才会设置此部分。
- `[P<domain_number>]p<bus>s<slot>[f<function>][u<usb_port>][…​][c<config>][i<interface>]`：USB设备
    对于 USB 设备，hub 端口号的完整链由 hub 的端口号组成。如果名称大于最大值（15 个字符），则不会导出该名称。如果链中有多个 USB 设备，则 udev 会抑制 USB 配置描述符(c1)和 USB 接口描述符(i0)的默认值。

> biosdevname 规范的常见名称包括：em1、p3p4、p3p4_1、。。。

udev 设备管理器会根据以下方案生成设备名称：

|Scheme|描述|示例|
|:-:|-|:-:|
|1|设备名称包含固件或者 BIOS 提供的索引号，用于板上的设备。如果此信息不可用或不适用，则 udev 将使用方案 2。|eno1，主板集成的1号以太网卡|
|2|设备名称包含固件或 BIOS 提供的 PCI Express（PCIe）热插件插槽索引号。如果此信息不可用或不适用，则 udev 将使用方案 3。|ens1，内置1号PCI接口的以太网卡|
|3|设备名称包含硬件连接器的物理位置。如果此信息不可用或不适用，则 udev 将使用方案 5。|enp2s0，2号PCI扩展卡的slot 0的以太网卡|
|4|设备名称包含 MAC 地址。Red Hat Enterprise Linux 默认不使用这个方案，但管理员可选择性地使用它。|enx525400d5e0fb，网卡的MAC地址|
|5|传统的无法预计的内核命名方案。如果 udev 无法应用任何其他方案，则设备管理器使用这个方案。|eth0，传统方案|

更多示例包括：

- wlp3s0 ：PCI无线网卡
- wwp0s29f7u2i2 ：4G modem
- wlp0s2f1u4u1 ：连接在USB Hub上的无线网卡
- enx78e7d1ea46da ：直接以MAC地址命名的 PCI 网卡

---

## 参考文献

- [Red Hat 官方文档 - 一致的网络接口设备命名](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/consistent-network-interface-device-naming_configuring-and-managing-networking)
- [linux 目录/sys 解析](https://www.cnblogs.com/jikexianfeng/p/9209663.html)
- [Centos 网卡命名规范及信息查看](https://cloud.tencent.com/developer/article/1694844)
- [biosdevname网卡命名方式](https://www.cnblogs.com/jackydalong/archive/2013/11/06/3410890.html)
- [CentOS 7 下网络管理之命令行工具nmcli](https://www.jianshu.com/p/5d5560e9e26a)
- [从 NetworkManager 切换到 Systemd-networkd](https://linux.cn/article-6629-1.html)
- [Netplan Github 主页](https://github.com/canonical/netplan)
  