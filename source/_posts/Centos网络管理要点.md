---
title: Centos网络管理要点
date: 2023-08-19 10:58:31
tags:
---

## 一、网络接口

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

## 二、网络服务

### 1. network.service

Centos7 之前的版本都是通过 `network.service` 管理网络配置，控制脚本位于 `/etc/rc.d/init.d/network`，网卡配置文件位于`/etc/sysconfig/network-scripts`。

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

`network.service` 是一个非常简单的软件包（仅有264行代码的脚本文件），其设计目标时非常简单的有线网络环境，当网络接口配置信息修改后，网络服务必须重新启动，因此很难满足无线网络切换的需求。

### 2. NetworkManager.service

RedHat 在2004年启动了 NetworkManager 项目，皆在能够让Linux用户更轻松的处理现代网络需求，尤其是无线网络，能够自动发现网卡并配置IP地址，现在由 GNOME 管理，有一个的漂亮的客户端界面 nmtui。
Centos7 同时支持 network.service 和 NetworkManager.service，相当于在 Centos7 的一个过渡，默认情况下这2个服务都有开启，但是因为 NetworkManager.service 当时的兼容性不好，大部分人都会将其关闭。

Centos 8 已经废弃 network.service（默认不安装），只能通过 NetworkManager 进行网络配置，OpenEuler 21.10 也是这样。

systemd-networkd和NetworkManager都是用于管理Linux系统网络配置的工具，它们之间的区别在于设计目标和实现方式。

### 3. systemd-networkd.service

systemd-networkd是Systemd计划的一部分，旨在提供一个轻量级、高性能的网络管理器。它使用原生Linux网络接口，不依赖第三方软件包，因此占用更少的资源和内存。systemd-networkd通过systemd网络守护进程启动和管理，可以实现自动启动、监控和重启，提高了系统的可靠性和稳定性。

相比之下，NetworkManager则更加全面和复杂，设计目标是提供更多的功能和管理选项。它使用DBus作为通信机制，可以集成各种网络类型和设备，包括以太网、无线网络、蓝牙、移动宽带等。NetworkManager支持广泛的网络协议和语言环境，并提供图形界面和命令行界面等多种管理方式。

综上所述，systemd-networkd更适合轻量级、嵌入式或服务器环境，优势在于快速启动、低内存占用和高性能；而NetworkManager则更适合桌面环境和对网络性能和安全性要求更高的场景，优势在于功能全面、易于管理和配置。

需要注意的是，这两种网络管理工具并不互斥，可以根据需要同时使用。在Ubuntu 18.04及更高版本中，默认使用systemd-networkd作为网络管理器，但也可以选择切换到NetworkManager。

> 在 libvirtd 没有虚拟机运行时，拔网线后，上面自动分配的ip会被 systemd-networkd 清理掉， 但如果 libvirtd 有虚拟机正在运行，那么拔网线后，br0 上面的ip不会清理，路由仍然存在， 这导致拔网线后，不能自动切换到 wlan0 运行，因为br0的路由优先级比较高。

systemd 的其中一部分是 systemd-networkd，它负责 systemd 生态中的网络配置。使用 systemd-networkd，你可以为网络设备配置基础的 DHCP/静态 IP 网络。它还可以配置虚拟网络功能，例如网桥、隧道和 VLAN。systemd-networkd 目前还不能直接支持无线网络，但你可以使用 wpa_supplicant 服务配置无线适配器，然后把它和 systemd-networkd 联系起来。

在很多 Linux 发行版中，NetworkManager 仍然作为默认的网络配置管理器。和 NetworkManager 相比，systemd-networkd 仍处于积极的开发状态，还缺少一些功能。例如，它还不能像 NetworkManager 那样能让你的计算机在任何时候通过多种接口保持连接。它还没有为更高层面的脚本编程提供 ifup/ifdown 钩子函数。但是，systemd-networkd 和其它 systemd 组件（例如用于域名解析的 resolved、NTP 的timesyncd，用于命名的 udevd）结合的非常好。随着时间增长，systemd-networkd只会在 systemd 环境中扮演越来越重要的角色。

### 3. 小结

systemd-networkd和NetworkManager都是用于管理Linux系统网络配置的工具，它们之间的区别在于设计目标和实现方式。

systemd-networkd是Systemd计划的一部分，旨在提供一个轻量级、高性能的网络管理器。它使用原生Linux网络接口，不依赖第三方软件包，因此占用更少的资源和内存。systemd-networkd通过systemd网络守护进程启动和管理，可以实现自动启动、监控和重启，提高了系统的可靠性和稳定性。

相比之下，NetworkManager则更加全面和复杂，设计目标是提供更多的功能和管理选项。它使用DBus作为通信机制，可以集成各种网络类型和设备，包括以太网、无线网络、蓝牙、移动宽带等。NetworkManager支持广泛的网络协议和语言环境，并提供图形界面和命令行界面等多种管理方式。

综上所述，systemd-networkd更适合轻量级、嵌入式或服务器环境，优势在于快速启动、低内存占用和高性能；而NetworkManager则更适合桌面环境和对网络性能和安全性要求更高的场景，优势在于功能全面、易于管理和配置。

需要注意的是，这两种网络管理工具并不互斥，可以根据需要同时使用。在Ubuntu 18.04及更高版本中，默认使用systemd-networkd作为网络管理器，但也可以选择切换到NetworkManager。

---

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

asd 

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

#### network.service

在 Centos 7 的版本，通过 network 提供基础网络管理服务，
注意，network 控制，来激活网络新配置，从而使得配置生效。

#### systemd-networkd.service

#### Network-Manager.service

Network-Manager 最初由 Redhat 公司开发，主要目标是自动检测网络环境，自由切换在线和离线模式。
Network-Manager 优先选择有线网络，并支持无线网络的自动切换，。


---

## 附录一：网卡设备的命名方式

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


devtmpfs 的作用是在 linux 核心启动早期建立一个初步的 /dev ;令一般启动程序不用等待 udev，从而缩短 GNU/Linux 的开机时间。
udev 是Linux kernel 2.6系列的设备管理器。它主要的功能是管理/dev目录底下的设备节点

```console
[root@MiWiFi-RA70-srv /]# df -h
文件系统        容量  已用  可用 已用% 挂载点
devtmpfs        219M     0  219M    0% /dev
tmpfs           235M     0  235M    0% /dev/shm
tmpfs           235M  3.4M  232M    2% /run
tmpfs           235M     0  235M    0% /sys/fs/cgroup
/dev/sda1        10G  2.6G  7.5G   26% /
tmpfs           235M     0  235M    0% /tmp
tmpfs            47M     0   47M    0% /run/user/0

[root@MiWiFi-RA70-srv /]# free -h
              total        used        free      shared  buff/cache   available
Mem:          469Mi       103Mi       177Mi       3.0Mi       188Mi       103Mi
Swap:            0B          0B          0B
```

proc：虚拟文件系统，在linux系统中被挂载与/proc目录下。里面的文件包含了很多系统信息，比如cpu负载、 内存、网络配置和文件系统等等。我们可以通过内部文本流来查看进程信息（正在运行的各个进程的PID号也以目录名形式存在/proc目录下）和机器的状态。
tmpfs：虚拟内存文件系统，使用内存作为临时存储分区，掉电之后会丢失数据，创建时不需要使用mkfs等格式化
devfs:设备文件，提供类似于文件的方法来管理位于/dev目录下的设备
sysfs：虚拟内存文件系统，2.6内核之前没有规定sysfs的标准挂载目录，但是在2.6之后就规定了要挂载到/sys目录下（针对以前的 sysfs 挂载位置不固定或没有标准被挂载，有些程序从 /proc/mounts 中解析出 sysfs 是否被挂载以及具体的挂载点，这个步骤现在已经不需要了）。它的作用类似于proc，但除了与 proc 相同的具有查看和设定内核参数功能之外，还有为 Linux 统一设备模型作为管理之用。相比于 proc 文件系统，使用 sysfs 导出内核数据的方式更为统一，并且组织的方式更好。

与proc的比较：

sysfs 与 proc 相比有很多优点，最重要的莫过于设计上的清晰。一个 proc 虚拟文件可能有内部格式，如 /proc/scsi/scsi ，它是可读可写的，(其文件权限被错误地标记为了 0444 ！，这是内核的一个BUG)，并且读写格式不一样，代表不同的操作，应用程序中读到了这个文件的内容一般还需要进行字符串解析，而在写入时需要先用字符串格式化按指定的格式写入字符串进行操作；相比而言， sysfs 的设计原则是一个属性文件只做一件事情， sysfs 属性文件一般只有一个值，直接读取或写入。整个 /proc/scsi 目录在2.6内核中已被标记为过时(LEGACY)，它的功能已经被相应的 /sys 属性文件所完全取代。新设计的内核机制应该尽量使用 sysfs 机制，而将 proc 保留给纯净的“进程文件系统”。

--

sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
devtmpfs on /dev type devtmpfs (rw,nosuid,size=224160k,nr_inodes=56040,mode=755)
securityfs on /sys/kernel/security type securityfs (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000)
tmpfs on /run type tmpfs (rw,nosuid,nodev,mode=755)
tmpfs on /sys/fs/cgroup type tmpfs (ro,nosuid,nodev,noexec,mode=755)
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
pstore on /sys/fs/pstore type pstore (rw,nosuid,nodev,noexec,relatime)
bpf on /sys/fs/bpf type bpf (rw,nosuid,nodev,noexec,relatime,mode=700)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/rdma type cgroup (rw,nosuid,nodev,noexec,relatime,rdma)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls,net_prio)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/files type cgroup (rw,nosuid,nodev,noexec,relatime,files)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
configfs on /sys/kernel/config type configfs (rw,nosuid,nodev,noexec,relatime)
/dev/sda1 on / type xfs (rw,relatime,attr2,inode64,noquota)
systemd-1 on /proc/sys/fs/binfmt_misc type autofs (rw,relatime,fd=33,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=16510)
mqueue on /dev/mqueue type mqueue (rw,nosuid,nodev,noexec,relatime)
hugetlbfs on /dev/hugepages type hugetlbfs (rw,relatime,pagesize=2M)
debugfs on /sys/kernel/debug type debugfs (rw,nosuid,nodev,noexec,relatime)
tmpfs on /tmp type tmpfs (rw,nosuid,nodev)
tmpfs on /run/user/0 type tmpfs (rw,nosuid,nodev,relatime,size=48088k,mode=700)


/run/user/$uid由pam_systemd该用户创建并用于存储该用户正在运行的进程使用的文件。这些可能是诸如密钥环守护程序，pulseaudio等之类的东西。

在系统化之前，这些应用程序通常将其文件存储在中/tmp。/home/$user由于主目录通常通过网络文件系统挂载，因此它们不能使用位置，并且这些文件不应在主机之间共享。/tmp是FHS指定的唯一本地且可被所有用户写入的位置。

但是，将所有这些文件存储在每个人都可以写的文件中/tmp是有问题的/tmp，尽管您可以更改正在创建的文件的所有权和模式，但使用起来更加困难。

于是systemd诞生了/run/user/$uid。该目录在系统本地，并且只能由目标用户访问。因此，希望将文件存储在本地的应用程序不再需要担心访问控制。
它还可以使事情保持井井有条。当用户注销并且没有剩余活动会话时，pam_systemd将清除该/run/user/$uid目录。由于周围散布着各种文件/tmp，您无法执行此操作。

--

sysfs 具体包含三种类型：

- 针对进程信息的 proc 文件系统
- 针对设备的 devfs 文件系统
- 针对伪终端的 devpts 文件系统

## 二、网络服务

---

## 参考文献

- [Red Hat 官方文档 - 一致的网络接口设备命名](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/consistent-network-interface-device-naming_configuring-and-managing-networking)
- [linux 目录/sys 解析](https://www.cnblogs.com/jikexianfeng/p/9209663.html)
- [Centos 网卡命名规范及信息查看](https://cloud.tencent.com/developer/article/1694844)
- [biosdevname网卡命名方式](https://www.cnblogs.com/jackydalong/archive/2013/11/06/3410890.html)