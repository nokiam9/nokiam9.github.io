---
title: Ubuntu 18.04.4的安装步骤
date: 2020-06-20 16:10:49
tags:
---

这几天折腾NUC主机，在安装Ubuntu 18.04.4 LTS（长期支持的稳定版本）的过程中遇到不少坑，立此存照吧。

## Step 0：安装前的准备工作

已安装内存和硬盘的NUC主机之外，还需要：

- 独立显示器：感谢LP赞助的DELL S2319SP

- USB键盘 + USB鼠标：开机BIOS设置硬件启动顺序时键盘操作不方便

- 有线网络接口：备用先不接

- Ubuntu启动U盘：制作方法参见[如何制作Ubuntu的USB启动盘](如何制作Ubuntu的USB启动盘.md)，Server版本912M，Desktop版本2.13G

## Step 1: 安装Ubuntu Server

1. 插入Ubuntu安装U盘，开机启动后进入安装界面，各种参数选默认

2. 完成安装后拔出U盘，重新启动并登录进入新安装的Ubuntu Server

>- 开机启动过程中，可能需要在BIOS设置启动顺序（USB---STAT---LAN），进入方法是加电过程中持续按F2
>- 安装过程中需要设置一个默认用户，root尚未激活  
>- 由于安装过程中未接网线，完成后所有网络接口都不可用

{% asset_img intel-bios.png %}

## step 2: 激活root用户

1. 以默认用户身份设置root口令，并切换到root

    ``` cmd
    sj@nuc5i3:~$ sudo passwd root
    [sudo] password for sj:
    Enter new UNIX password:
    Retype new UNIX password:
    passwd: password updated successfully
    sj@nuc5i3:~$ su -
    Password:
    root@nuc5i3:~#
    ```

2. 如果需要允许root用户远程登录，修改配置文件`/etc/ssh/sshd_config`，将`PermitRootLogin`改为`yes`

    ``` txt
    #PermitRootLogin prohibit-password
    ```

## Step 3: 设置有线网络接入

1. 编辑网卡配置文件`/etc/netplan/50-cloud-init.yaml`，至少需要设置IP地址和默认网关

    ``` yaml
    # This file is generated from information provided by the datasource.  Changes
    # to it will not persist across an instance reboot.  To disable cloud-init's
    # network configuration capabilities, write a file
    # /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
    # network: {config: disabled}
    network:
        version: 2

        ethernets:
            enp0s25:
                dhcp4: no
                addresses: [192.168.0.130/24]
                gateway4: 192.168.0.1
    ```

2. 插入有线网络，`netpaln generate`刷新网络配置，`netplan apply`启动有线网卡

3. 检查网络状态，`ifconfig`确认有线网卡`enp0s25`启动成功

    ``` cmd
    root@nuc5i3:/etc/netplan# netplan generate
    root@nuc5i3:/etc/netplan# netplan apply
    root@nuc5i3:/etc/netplan# ifconfig
    enp0s25: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
            inet 192.168.0.130  netmask 255.255.255.0  broadcast 192.168.0.255
            inet6 fe80::baae:edff:fe73:87fb  prefixlen 64  scopeid 0x20<link>
            ether b8:ae:ed:73:87:fb  txqueuelen 1000  (Ethernet)
            RX packets 23068  bytes 14045959 (14.0 MB)
            RX errors 0  dropped 23  overruns 0  frame 0
            TX packets 12173  bytes 1273580 (1.2 MB)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
            device interrupt 20  memory 0xf7100000-f7120000  

    lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
            inet 127.0.0.1  netmask 255.0.0.0
            inet6 ::1  prefixlen 128  scopeid 0x10<host>
            loop  txqueuelen 1000  (Local Loopback)
            RX packets 144  bytes 13680 (13.6 KB)
            RX errors 0  dropped 0  overruns 0  frame 0
            TX packets 144  bytes 13680 (13.6 KB)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
    ```

## Step 4: 设置Wifi网络接入（可选）

1. 在确认已连接公网的条件下，在线安装`network-manager`（其中包括了无线网卡驱动程序等必需的软件包）

   ``` cmd
   # apt update
   # apt install network-manager
   ```

2. 再次编辑`/etc/netplan/50-cloud-init.yaml`，至少需要配置`Access-points`的SSID和接入密码等。
   注意：必须显示定义渲染方式为`NetworkManager`，这也是上一步需要apt安装的原因。

    ``` yaml
    # This file is generated from information provided by the datasource.  Changes
    # to it will not persist across an instance reboot.  To disable cloud-init's
    # network configuration capabilities, write a file
    # /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
    # network: {config: disabled}
    network:
        version: 2
        renderer: NetworkManager

        ethernets:
            enp0s25:
                dhcp4: no
                addresses: [192.168.0.130/24]
                gateway4: 192.168.0.1

        wifis:
            wlp2s0:
                access-points:
                    "NETGEAR59":
                        password: "xxxxxxxx"
                dhcp4: no
                addresses: [192.168.0.131/24]
                gateway4: 192.168.0.1
    ```

3. 再次`netplan generate`检查并刷新网络配置，`netplan apply`启动无线网卡。
   注意：`netplan`有中间配置文件的刷新机制，generate + apply是一个良好的操作习惯。

4. 检查网络状态，`ifconfig`确认无线网卡`wlp2s0`启动成功

    ``` cmd
    root@nuc5i3:/etc/netplan# netplan generate
    root@nuc5i3:/etc/netplan# netplan apply
    root@nuc5i3:/etc# ifconfig
    ......
    wlp2s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
            inet 192.168.0.131  netmask 255.255.255.0  broadcast 192.168.0.255
            inet6 fe80::3613:e8ff:fe25:20d8  prefixlen 64  scopeid 0x20<link>
            ether 34:13:e8:25:20:d8  txqueuelen 1000  (Ethernet)
            RX packets 501  bytes 94281 (94.2 KB)
            RX errors 0  dropped 0  overruns 0  frame 0
            TX packets 44  bytes 4436 (4.4 KB)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
    ```

## Step 5: 设置DNS服务

---

## 附录一：关于硬盘分区表

### 名词解释

磁盘管理模式：

- MBR（Master Boot Record）：即硬盘的主引导记录分区列表，在主引导扇区，位于硬盘的cylinder 0， head 0， sector 1 （Sector是从1开始的）。

- GPT（GUID Partition Table）：即全局唯一标识分区列表，是一个物理硬盘的分区结构。它用来替代BIOS中的主引导记录分区表（MBR）。

主板接口标准：

- BIOS（Basic Input Output System）：它的全称应该是ROM－BIOS，意思是只读存储器基本输入输出系统。其实，它是一组固化到计算机内主板上一个ROM芯片上的程序，它保存着计算机最重要的基本输入输出的程序、系统设置信息、开机上电自检程序和系统启动自举程序。 其主要功能是为计算机提供最底层的、最直接的硬件设置和控制。

- UEFI（Unified Extensible Firmware Interface)：全称“统一的可扩展固件接口”， 是一种详细描述全新类型接口的标准。这种接口用于操作系统自动从预启动的操作环境，加载到一种操作系统上，从而使开机程序化繁为简，节省时间。

### 技术分析

MBR是传统的分区表类型，最大的缺点则是不支持容量大于2T的硬盘。GPT则弥补了MBR这个缺点，最大支持18EB的硬盘，是基于UEFI使用的磁盘分区架构。

传统BIOS主要支持MBR引导，UEFI则是取代传统BIOS，它加入了对新硬件的支持，其中就有2TB以上硬盘。

目前所有Windows系统均支持MBR，而GPT只有64位系统才能支持。BIOS只支持MBR引导系统，而GPT仅可用UEFI引导系统。正因为这样，现在主板大多采用BIOS集成UEFI，或UEFI集成BIOS，以此达到同时兼容MBR和GPT引导系统的目的。

UEFI启动引导系统的方法是查找硬盘分区中第一个FAT分区内的引导文件进行系统引导，这里并无指定分区表格式。所以U盘和移动硬盘可以用MBR分区表，创建一个FAT分区放置引导文件，从而达到可以双模式启动的目的。但需要注意的是，UEFI虽然支持MBR启动，但必须要有UEFI引导文件存放在FAT分区下；UEFI是无法使用传统MBR引导来启动系统的。

由于GPT引导系统的方式与MBR不同，故而使用传统系统安装办法（如Ghost、Win$Man等）会导致出现系统无法引导的现象。而且使用GPT引导系统的话，必要时还得调整主板设置，开启UEFI（大部分主板默认开启UEFI）。但是使用UEFI和GPT，只是支持大于容量2T的硬盘，并不会带来质的提升（开机硬件自检会稍微快了那么1、2秒）。所以，如果不用大于2T的硬盘做系统的话，就没必要使用UEFI。

- BIOS+MBR：这是最传统的，系统都会支持；唯一的缺点就是不支持容量大于2T的硬盘。

- BIOS+GPT：BIOS是可以使用GPT分区表的硬盘来作为资料盘的，但不能引导系统；若电脑同时带有容量小于2T的硬盘和容量大于2T的硬盘，小于2T的可以用MBR分区表安装系统，而大于2T的可以使用GPT分区表来存放资料。但系统须使用64位系统。

- UEFI+MBR：可以把UEFI设置成Legacy模式（传统模式）让其支持传统MBR启动，效果同BIOS+MBR；也可以建立FAT分区，放置UEFI启动文件来，可应用在U盘和移动硬盘上实现双模式启动。

- UEFI+GPT：如果要把大于2T的硬盘作为系统盘来安装系统的话，就必须如此。而且系统须使用64位系统，否则无法引导。但系统又不是传统在PE下安装后就能直接使用的，引导还得经过处理才行。

### 实际案例

Ubuntu安装完成后，主机硬盘的默认分区表为GPT格式（不是MBR格式），包含了两个分区，其中：

- `/dev/sda1`：用于EFI启动，512M
- `/dev/sda2`：Linunx系统盘

``` sh
root@nuc5i3:~# fdisk -l
Disk /dev/loop0: 89.1 MiB, 93417472 bytes, 182456 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sda: 111.8 GiB, 120034123776 bytes, 234441648 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: D8D94899-BC23-4D81-96EB-ECAFE4710A7E

Device       Start       End   Sectors   Size Type
/dev/sda1     2048   1050623   1048576   512M EFI System
/dev/sda2  1050624 234438655 233388032 111.3G Linux filesystem
```

## 附录二： 如何查看各类网卡的逻辑设备名

不同型号的硬件设备，由于其驱动程序的差异，在Ubuntu安装完成后会有不同的设备名。

使用`lshw`命令，可以查询全部硬件设备信息，并找到其逻辑设备名。

``` cmd
# lshw
nuc5i3
    description: Desktop Computer
    width: 64 bits
    capabilities: smbios-2.8 dmi-2.8 smp vsyscall32
    configuration: boot=normal chassis=desktop uuid=00EC2C62-7872-E311-B04F-B8AEED7387FB
......
        *-network
             description: Ethernet interface
             product: Ethernet Connection (3) I218-V
             vendor: Intel Corporation
             physical id: 19
             bus info: pci@0000:00:19.0
             logical name: enp0s25
             version: 03
             serial: b8:ae:ed:73:87:fb
             size: 100Mbit/s
             capacity: 1Gbit/s
             width: 32 bits
             clock: 33MHz
......
        *-network
            description: Wireless interface
            product: Wireless 7265
            vendor: Intel Corporation
            physical id: 0
            bus info: pci@0000:02:00.0
            logical name: wlp2s0
            version: 59
            serial: 34:13:e8:25:20:d8
            width: 64 bits
            clock: 33MHz
......
```

## 附录三： 关于DNS Server
