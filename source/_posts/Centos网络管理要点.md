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