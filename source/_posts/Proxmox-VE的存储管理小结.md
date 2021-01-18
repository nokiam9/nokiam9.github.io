---
title: Proxmox-VE的存储管理小结
date: 2021-01-17 23:47:09
tags:
---

## PVE的基本存储类型

与通常的UNIX系统一样，Proxmox VE将存储分为两种:

- 基于POSIX的文件系统存储。可以存储所有类型的文件，但是性能一般。
- 基于RAW裸设备的块存储设备。由于块存储没有文件概念，因此只能存储VM镜像和Disk数据盘，注定无法存储ISO镜像文件、Backup备份文件、Snapshot快照文件等。

各种存储方式的特性见下表。

{% asset_img proxmox-storage-types.png %}

PVE将后端存储的配置文件存放在`/etc/pve/storage.cfg`中，具体信息为：

``` console
root@pve01:~# cat /etc/pve/storage.cfg
dir: local
    path /var/lib/vz
    content vztmpl,iso,backup

lvmthin: local-lvm
    thinpool data
    vgname pve
    content images,rootdir

nfs: nfs130
    export /data/nfs
    path /mnt/pve/nfs130
    server 192.168.0.130
    content images
```

下面介绍几个重要的存储类型：

## 目录（Directory）

Proxmox VE 可以使用本地目录或挂载在本地文件系统的共享存储作为存储服务。
目录是文件系统级的存储服务，你可以保存任何类型的数据，包括虚拟机镜像，容器，模板， ISO 镜像或虚拟机备份文件。
PVE初始安装生成的`local`存储就是`Directory`属性，其路径是`/var/lib/vz`。

{% asset_img shot2.png %}

可以看到，其中`dump`目录存储的就是备份文件，`template`目录存储的就是VM模版和LXC模版。

``` console
root@pve01:/var/lib/vz# tree /var/lib/vz
/var/lib/vz
├── dump
│   ├── vzdump-qemu-120-2021_01_18-00_17_53.log
│   └── vzdump-qemu-120-2021_01_18-00_17_53.vma.zst
├── images
└── template
    ├── cache
    │   ├── alpine-3.11-default_20200425_amd64.tar.xz
    │   ├── centos-7-default_20190926_amd64.tar.xz
    │   └── ubuntu-18.04-standard_18.04.1-1_amd64.tar.gz
    ├── iso
    │   ├── CentOS-7-x86_64-DVD-2003.iso
    │   ├── CentOS-7-x86_64-Minimal-2003.iso
    │   └── ubuntu-18.04.4-live-server-amd64.iso
    └── qemu
```

## LVM

LVM 是典型的块存储解决方案，但 LVM 后端存储本身不支持快照和链接克隆功能。更不幸的是，在创建普通 LVM 快照期间，整个卷组的写操作都会受到影响而变得非常低效。

> LVM最大的好处是你可以在共享存储上建立 LVM 后端存储服务。例如可以在 iSCSI LUN 上建立 LVM。LVM 后端存储自带 Proxmox VE 集群锁以有效防止并发访问冲突。

LVM的创建包含了以下步骤： PV -> VG -> LV，具体步骤参见附录1。

{% asset_img shot4.png %}

## 薄存储（LVM-thin）

LVM 是在逻辑卷创建时就按设置的卷容量大小预先分配所需空间。LVM-thin 存储池是在向 卷内写入数据时按实际写入数据量大小分配所需空间。LVM-thin 所用的存储空间分配方式允许创建容量远大于物理存储空间的存储卷，因此也称为“薄模式”。

> 注意：LVM-thin 存储池不能被多个节点同时共享使用，只能用于节点本地存储.

创建和管理 LVM-thin 存储池的命令和 LVM 命令完全一致(参见 man lvmthin)。假定你已 经有一个 LVM 卷组 pve，如下命令可以创建一个名为 data 的新 LVM-thin 存储池(容量 100G):

``` shell
lvcreate -L 100G -n data pve
lvconvert --type thin-pool pve/data
```

也可以在PVE的GUI界面进行操作。

{% asset_img shot4.png %}

## 网络文件系统（NFS）

基于 NFS 的后端存储服务实际上建立在目录后端存储之上，其属性和目录后端存储非常相似。其中子目录布局和文件命名规范完全一致。
NFS 后端存储的优势在于，你可以通过配置 NFS 服务器参数，实现 NFS 存储服务自动挂载，而无需编辑修改/etc/fstab 文件。

NFS 存储服务能够自动检测 NFS 服务器的在线状态，并自动连接 NFS 服务器输出的共享存储服务。

## CIFS

CIFS（Common Internet File System）就是 SMB 的改进版本。Windows的文件共享其实就是使用了 SMB或者说 CIFS。
基于 CIFS 的后端存储可用于扩展基于目录的存储，这样就无需再手工配置 CIFS 挂载。该类 型存储可直接通过 Proxmox VE API 或 WebUI 添加。服务器心跳检测或共享输出选项等后端 存储参数配置也将自动完成配置。

---

## 附录1：安装LVM2

Centos基本安装包并不包含LVM管理功能，需用`yum install lvm2`自行安装。

- PV（Physical Volume）- 物理卷
    物理卷在逻辑卷管理中处于最底层，它可以是实际物理硬盘上的分区，也可以是整个物理硬盘，也可以是raid设备。
- VG（Volumne Group）- 卷组
    卷组建立在物理卷之上，一个卷组中至少要包括一个物理卷，在卷组建立之后可动态添加物理卷到卷组中。一个逻辑卷管理系统工程中可以只有一个卷组，也可以拥有多个卷组。
- LV（Logical Volume）- 逻辑卷
    逻辑卷建立在卷组之上，卷组中的未分配空间可以用于建立新的逻辑卷，逻辑卷建立后可以动态地扩展和缩小空间。系统中的多个逻辑卷可以属于同一个卷组，也可以属于不同的多个卷组。

{% asset_img lvm2.png %}

在安装lvm2后，就可以使用以下命令了。

``` console
root@pve01:~# pvs
  PV           VG           Fmt  Attr PSize    PFree  
  /dev/nvme0n1 date-nvme0n1 lvm2 a--  <476.94g 124.00m
  /dev/sda3    pve          lvm2 a--  <111.29g  13.87g
root@pve01:~# vgs
  VG           #PV #LV #SN Attr   VSize    VFree  
  date-nvme0n1   1   1   0 wz--n- <476.94g 124.00m
  pve            1  24   0 wz--n- <111.29g  13.87g
root@pve01:~# lvs
  LV                     VG           Attr       LSize    Pool Origin          Data%  Meta%  Move Log Cpy%Sync Convert
  date-nvme0n1           date-nvme0n1 twi-aotz-- <467.28g                      0.00   0.37                            
  base-120-disk-0        pve          Vri-a-tz-k    4.00g data                 38.28                                  
  base-121-disk-0        pve          Vri-a-tz-k    4.00g data                 38.53                                  
  base-122-disk-0        pve          Vri-a-tz-k    8.00g data                 54.14                                  
  data                   pve          twi-aotz--   59.66g                      42.54  2.87                            
  root                   pve          -wi-ao----   27.75g                                                             
  snap_vm-199-disk-0_dns pve          Vri---tz-k    4.00g data vm-199-disk-0                                          
  swap                   pve          -wi-ao----    8.00g                                                             
  vm-100-cloudinit       pve          Vwi-a-tz--    4.00m data                 9.38                                   
  vm-100-disk-0          pve          Vwi-a-tz--   12.00g data                 59.46                                  
  vm-120-cloudinit       pve          Vwi-a-tz--    4.00m data                 9.38                                   
  vm-121-cloudinit       pve          Vwi-a-tz--    4.00m data                 9.38                                   
  vm-122-cloudinit       pve          Vwi-a-tz--    4.00m data                 9.38                                   
  vm-160-disk-0          pve          Vwi-aotz--    8.00g data                 7.74                                   
  vm-198-cloudinit       pve          Vwi-aotz--    4.00m data                 9.38                                   
  vm-198-disk-0          pve          Vwi-aotz--    4.00g data                 38.55                                  
  vm-199-cloudinit       pve          Vwi-aotz--    4.00m data                 9.38                                   
  vm-199-disk-0          pve          Vwi-aotz--    4.00g data                 38.56                                  
  vm-199-state-dns       pve          Vwi-a-tz--   <1.49g data                 33.03                                  
  vm-200-cloudinit       pve          Vwi-aotz--    4.00m data                 9.38                                   
  vm-200-disk-0          pve          Vwi-aotz--    4.00g data                 38.58                                  
  vm-222-cloudinit       pve          Vwi-a-tz--    4.00m data                 9.38                                   
  vm-222-disk-0          pve          Vwi-a-tz--    4.00g data                 54.27                                  
  vm-223-cloudinit       pve          Vwi-aotz--    4.00m data                 9.38                                   
  vm-223-disk-0          pve          Vwi-aotz--    8.00g data base-122-disk-0 85.10 
root@pve01:~# lsblk
NAME                            MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda                               8:0    0 111.8G  0 disk 
├─sda1                            8:1    0  1007K  0 part 
├─sda2                            8:2    0   512M  0 part /boot/efi
└─sda3                            8:3    0 111.3G  0 part 
  ├─pve-swap                    253:0    0     8G  0 lvm  [SWAP]
  ├─pve-root                    253:1    0  27.8G  0 lvm  /
  ├─pve-data_tmeta              253:2    0     1G  0 lvm  
  │ └─pve-data-tpool            253:4    0  59.7G  0 lvm  
  │   ├─pve-data                253:5    0  59.7G  0 lvm  
  │   ├─pve-vm--198--cloudinit  253:7    0     4M  0 lvm  
  │   ├─pve-vm--198--disk--0    253:8    0     4G  0 lvm  
  │   ├─pve-vm--199--disk--0    253:9    0     4G  0 lvm  
  │   ├─pve-vm--199--cloudinit  253:10   0     4M  0 lvm  
  │   ├─pve-vm--121--cloudinit  253:11   0     4M  0 lvm  
  │   └─pve-base--121--disk--0  253:26   0     4G  1 lvm  
  └─pve-data_tdata              253:3    0  59.7G  0 lvm  
    └─pve-data-tpool            253:4    0  59.7G  0 lvm  
      ├─pve-data                253:5    0  59.7G  0 lvm  
      ├─pve-vm--198--cloudinit  253:7    0     4M  0 lvm  
      ├─pve-vm--198--disk--0    253:8    0     4G  0 lvm  
      ├─pve-vm--199--disk--0    253:9    0     4G  0 lvm  
      ├─pve-vm--199--cloudinit  253:10   0     4M  0 lvm  
      ├─pve-vm--121--cloudinit  253:11   0     4M  0 lvm  
      ├─pve-vm--199--state--dns 253:17   0   1.5G  0 lvm  
      └─pve-base--121--disk--0  253:26   0     4G  1 lvm  
nvme0n1                         259:0    0   477G  0 disk 
├─data02-data02_tmeta           253:23   0   4.8G  0 lvm  
│ └─data02-data02               253:28   0 467.3G  0 lvm  
└─data02-data02_tdata           253:27   0 467.3G  0 lvm  
  └─data02-data02               253:28   0 467.3G  0 lvm  
```

---

## 参考文献

- [Proxmox VE 安装的系列教程](https://blog.51cto.com/6222666/2161799)
- [Proxmox虚拟系统PVE的磁盘分区及文件系统分析总结](https://zhuanlan.zhihu.com/p/145862221)
- [CentOS7 LVM与RAID简单使用](https://blog.csdn.net/u011069498/article/details/96303220)
- [Proxmox 安装和设置](http://einverne.github.io/post/2020/03/proxmox-install-and-setup.html)
- [ProXmoX VE 安装及基础配置](https://post.smzdm.com/p/768830/)