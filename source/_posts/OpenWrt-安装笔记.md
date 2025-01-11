---
title: OpenWrt 安装笔记
date: 2025-01-11 18:42:26
tags:
---

## 一、概述

[OpenWrt](https://openwrt.org) 是一个为嵌入式设备（通常是无线路由器）开发的高扩展度的 GNU/Linux 发行版。与许多其他路由器的发行版不同，OpenWrt 是一个完全为嵌入式设备构建的功能全面、易于修改、由现代 Linux 内核驱动的操作系统。在实践中，这意味着您可以得到需要的所有功能，却仍能避免臃肿。

> LEDE（Linux Embedded Development Environment）曾经是 OpenWrt 的一个分支项目，2018 年合并后的项目使用 OpenWrt 的名字、LEDE的源代码。

### 1. 基线版本

OpenWrt 的版本序列参见：[https://downloads.openwrt.org/releases/](https://downloads.openwrt.org/releases/)，当前稳定版是 23.05，本文的基线版本是 2024 年 5 月发布的 23.05.3。

OpenWrt 是为嵌入式设备设计的，其支持的 CPU 架构非常广泛，甚至树莓派等低端设备都可以，本文采用 PVE 虚拟机安装，当然就是 x86-64 了，基础镜像为：[openwrt-23.05.3-x86-64-generic-ext4-combined.img](https://downloads.openwrt.org/releases/23.05.3/targets/x86/64)，采用 ext4 文件系统便于后续扩容。

> Squashfs 是一个**开源、只读、压缩**的文件系统，常被用于 Linux 发行版的 LiveCD 中。

### 2. opkg 包管理器

OpenWrt 的软件包管理工具是 opkg，功能与 dpkg 基本一致，其软件仓库大约有 3500 个包。
opkg 配置文件目录：`/etc/opkg/`

- `distfeeds.conf` ：默认软件源，系统安装时指向[https://downloads.openwrt.org](https://downloads.openwrt.org/releases/23.05.3/packages/x86_64)
- `customfeeds.conf`：用户软件源，初始为空。
- `keys/`：用于密钥管理

通常采用国内镜像源：

```config
src/gz openwrt_core https://mirrors.tuna.tsinghua.edu.cn/openwrt/releases/23.05.5/targets/x86/64/packages
src/gz openwrt_base https://mirrors.tuna.tsinghua.edu.cn/openwrt/releases/23.05.5/packages/x86_64/base
src/gz openwrt_luci https://mirrors.tuna.tsinghua.edu.cn/openwrt/releases/23.05.5/packages/x86_64/luci
src/gz openwrt_packages https://mirrors.tuna.tsinghua.edu.cn/openwrt/releases/23.05.5/packages/x86_64/packages
src/gz openwrt_routing https://mirrors.tuna.tsinghua.edu.cn/openwrt/releases/23.05.5/packages/x86_64/routing
src/gz openwrt_telephony https://mirrors.tuna.tsinghua.edu.cn/openwrt/releases/23.05.5/packages/x86_64/telephony
```

### 3. LuCI = Lua + UCI

UCI（Unified Configuration Interface）是 Openwrt 中为实现所有系统配置的统一配置接口。
Openwrt 的 LuCI 是一个采用 Lua 语言开发的 Web UI，基于 UCI 理念只包括一个精简的核心和最基本的库，体积小、启动速度快，可以实现路由的网页配置界面.

> LuCI 采用 MVC 三层架构，`/usr/lib/lua/luci/`下有三个目录`model`、`view`、`controller`

LuCI 的统一配置文件目录：`/etc/config/`，登录方式为 `http://<LAN IP>:80`。

```console
-rw-------    1 root     root           865 Jan 10 23:55 dhcp
-rw-------    1 root     root           134 Mar 23  2024 dropbear   # ssh 的替代品
-rw-r--r--    1 root     root          4066 Mar 23  2024 firewall   
-rw-r--r--    1 root     root           466 Jan 10 23:50 fstab      # 磁盘挂载点
-rw-r--r--    1 root     root           968 Jan  8 18:19 luci
-rw-------    1 root     root           342 Jan 10 23:47 network    # 网络配置文件
-rw-------    1 root     root           167 Mar 23  2024 rpcd
-rw-------    1 root     root           469 Jan 10 23:55 system     # 时区、时间服务器等        
-rw-r--r--    1 root     root           788 Mar 23  2024 ucitrack
-rw-------    1 root     root           783 Jan  8 18:19 uhttpd
```

## 二、基础系统安装

准备好基础镜像后，就可以创建一个虚拟机准备安装了。建议配置 1C + 1G、默认硬盘（无所谓，后面会删除）、默认 Lan 端口（因为是旁路无需增加 Wan 端口）。

1. 将基础镜像 SCP 到 node（OpenWrt默认不支持 sftp）的目录空间
2. 登录 node 的 Shell，加载基础镜像磁盘（展开后约 124 MB）
    `qm importdisk <VM ID> <Img Filename> local-lvm`
3. 删除 CDROM 和默认硬盘；加载镜像盘并**适当扩容**（建议 1GB）；
4. 注意检查 **Boot Order**，确保从镜像硬盘启动！
5. 现在启动 VM，如果一切正常就可以从 Console 登录了
6. 建议替换 opkg 的软件源为国内镜像，以加快后续安装速度

### 1. 网络配置

新系统初次启动后，通过 `ip a` 查看网络状态，默认状态为：

- lo：本地还回地址，127.0.0.1/8
- eth0：默认物理网口，实际指向`br-lan`
- br-lan：本地网桥接口，默认地址为`192.168.1.1`。

为了防止 IP 地址冲突，通常需要修改 `/etc/config/network` 配置文件：

```config
config interface 'lan'
    option device 'br-lan'
    option proto 'dhcp'
```

最后，就是重启网络服务了！

```bash
service network restart
```

### 2. 磁盘扩容

如果系统硬盘做了扩容，启动后磁盘信息如下：

```console
Device     Boot  Start     End Sectors  Size Id Type
/dev/sda1  *       512   33279   32768   16M 83 Linux
/dev/sda2        33792  246783  212992  104M 83 Linux
/dev/sda3       247808 2351103 2103296    1G 83 Linux
```

其中，`/dev/sda1` 是 Boot 引导盘，`/dev/sda2`是系统盘，`/dev/sda3`是新增但尚未使用的磁盘空间。

1. 安装工具软件：`opkg install parted fdisk block-mount blockdev`
2. 使用 `fdisk /dev/sda` 新增一个分区：m-帮助、p-列出、n-新增、w-存盘退出！
3. 使用 `mkfs.ext4 /dev/sda3` 将新分区格式化为 ext4 文件系统
4. 通过 overlay 文件系统，以**叠加**方式完成 root 文件系统扩容

```bash
DEVICE="/dev/sda3"
eval $(block info "${DEVICE}" | grep -o -e "UUID=\S*")
uci -q delete fstab.overlay
uci set fstab.overlay="mount"
uci set fstab.overlay.uuid="${UUID}"
uci set fstab.overlay.target="/overlay"
uci commit fstab
```

> uci 是 UCI 的命令行工具，提供 add set get show delete rename 等操作方法

换个角度，从 `/etc/config/fstab` 查看，其内容就是上述命令的处理结果。

```console
config global
    option anon_swap '0'
    option anon_mount '0'
    option auto_swap '1'
    option auto_mount '1'
    option delay_root '5'
    option check_fs '0'

config mount
    option target '/boot'
    option uuid '84173db5-fa99-e35a-95c6-28613cc79ea9'
    option enabled '0'

config mount
    option target '/'
    option uuid 'ff313567-e9f1-5a5d-9895-3ba130b4a864'
    option enabled '0'

config mount 'overlay'
    option uuid '70bbf914-80d7-455c-810a-fa19c6ed50a3'
    option target '/overlay'
```

最后，文件系统是如下形式，也可以通过 Web GUI 的 system - Mount Points 页面查看。

```console
Filesystem                Size      Used Available Use% Mounted on
/dev/root               102.3M     18.7M     81.5M  19% /rom
tmpfs                   493.2M      1.0M    492.2M   0% /tmp
/dev/sda3               976.3M      8.6M    900.4M   1% /overlay
overlayfs:/overlay      976.3M      8.6M    900.4M   1% /
/dev/sda1                15.7M      5.5M      9.8M  36% /boot
/dev/sda1                15.7M      5.5M      9.8M  36% /boot
tmpfs                   512.0K         0    512.0K   0% /dev
```

### 3. 新建磁盘

对于一个新增磁盘（例如`/dev/sdb`），基本的处理流程也类似，但注意目标磁盘不同！

1. 使用 `fdisk /dev/sdb` 新增一个分区：m-帮助、p-列出、n-新增、w-存盘退出！
2. 使用 `mkfs.ext4 /dev/sdb1` 将新分区格式化为 ext4 文件系统
3. 直接编辑 `/etc/config/fstab` 增加如下段落，然后重启即可！

```config
config mount
    option target '/mnt'
    option device '/dev/sdb1'
    option fstype ext4
    option options rw,sync
    option enabled 1
    option enable_fsck 0
```

> 这一步也可通过 Web GUI，在 system - Mount Points 页面的 Mount Points 段落的 Add 按钮进行编辑

### 4. 常用配置

OpenWrt 安装启动后，许多操作就可以通过 Web GUI 来操作了。

#### 启用系统时间服务

System - System 页面，点击 System Properties 段落的 Time Synchronization 标签：

- Enable NTP Client：开关
- Provide NTP Server：开关
- Bind NTP server：绑定 ntpd 服务的网络端口
- Use DHCP adertised servers：开关
- NTP Server candidates：上游时间服务器列表

#### 设置时区

System - System 页面，点击 System Properties 段落的 General Settings 标签：

- TimeZone：Aisa/Shanghai；点击 Save & Apply 生效

#### 关闭 DHCP 服务

Network - DHCP and DNS 页面，点击 CFG01411C 段落的 General 便签：

- Authoritative：取消勾选，不提供 DHCP 服务；点击 Save & Apply 生效

> 23.05.3 更新了 DHCP 的管理页，现在可以管理多个 Dnsmasq 实例
> CFG01411C 是缺省名称，查看 ps w ｜grep dnsmasq

## 三、常用软件安装

### 1. Transmission

- 安装应用软件包，可选中文提示：luci-i18n-transmission-zh-cn

```bash
opkg update
opkg install luci-app-transmission transmission-web transmission-cli
```

- 重启路由器！Web GUI 出现 Service - Transmission 页面，编辑如下内容：
  
  - Enabled：启用
  - Config file directory：/mnt/transmission。加挂磁盘后再次修改！
  - Run daemon as user：root。不再为权限烦恼
  - Run daemon as group：root
  - Incomplete directory enabled：默认关闭。没啥好处
  - Download directory：/mnt/transmission/done 。加挂磁盘后再次修改，也是以后的共享目录！
  - Cache size in MB：默认 2M

- 再次重启路由器！现在可以打开 Transmission 管理界面：`http://<LAN IP>:9091`
    现在下载数据目录`/mnt/transmission/` 的内容如下

```console
drwxr-xr-x    2 root     root          4096 Jan 12 02:29 blocklists
drwxr-xr-x    2 root     root          4096 Jan 12 02:29 done
drwxr-xr-x    2 root     root          4096 Jan 12 02:29 resume
-rw-r--r--    1 root     root          1355 Jan 12 02:29 settings.json
-rw-r--r--    1 root     root             0 Jan 12 02:29 stats.json
drwxr-xr-x    2 root     root          4096 Jan 12 02:29 torrents
```

- 下载需要大容量，后续为 VM 添加磁盘并挂载到 /mnt !!!

### 2. Samba4

- 安装应用软件包，可选中文提示：luci-i18n-samba4-zh-cn

```bash
opkg update
opkg install luci-app-samba4 samba4-admin samba4-client samba4-libs samba4-utils
```

如果是挂载其他硬盘，需要安装文件系统：

opkg install kmod-fs-ext4 kmod-fs-exfat kmod-fs-ntfs3 

这三个分别是ext4格式，exfat格式和ntfs格式硬盘的读写，按需即可

如果挂载是通过USB的(推荐至少要USB3.0接口)，还需要安装相关驱动和模块：

opkg install kmod-usb3 kmod-usb-storage-uas usbutils block-mount mount-utils luci-app-hd-idle
 
可选：

### 3. Frp

### 4. Passwall

---

## 附录一：瑞士军刀 busybox

BusyBox 被称为“嵌入式 Linux 的瑞士军刀”，设计目标就是在单一的可执行文件（不超过一张软盘）中提供精简的 Unix 工具集（200+ 应用程序），可运行于多款 POSIX 环境的操作系统。
查看 OpenWrt 的应用目录`\bin`，可以发现大部分的操作系统常用命令都是 busybox 提供的。

```console
-rwxr-xr-x    1 root     root        405522 Mar 23  2024 busybox
lrwxrwxrwx    1 root     root             7 Mar 23  2024 cat -> busybox
lrwxrwxrwx    1 root     root             7 Mar 23  2024 cp -> busybox
lrwxrwxrwx    1 root     root             7 Mar 23  2024 login -> busybox
lrwxrwxrwx    1 root     root             7 Mar 23  2024 ls -> busybox
-rwxr-xr-x    1 root     root        155131 Mar 23  2024 opkg
lrwxrwxrwx    1 root     root             7 Mar 23  2024 passwd -> busybox
```

## 附录二：service 系统服务管理

service 命令的执行过程实际上就是操作`/etc/init.d/`目录下的初始化脚本，这是传承自 System V 的标准风格。
值得一提的是，比`/etc/init.d/`优先级更高的是`/etc/rc[0..6].d/` 目录，其设定了不同级别的启动脚本，这也是 System V 的典型风格。当然，还有最高优先级的`/etc/inittab`脚本。

OpenWrt 的系统服务管理命令：

- `serive`：显示系统服务列表，包括服务名称、初始状态、运行状态

  ```console
  Usage: service <service> [command]
  /etc/init.d/boot                   enabled     stopped
  /etc/init.d/cron                   enabled     stopped
  /etc/init.d/dnsmasq                enabled     running
  /etc/init.d/log                    enabled     running
  /etc/init.d/network                enabled     running
  /etc/init.d/sysctl                 enabled     stopped
  /etc/init.d/system                 enabled     stopped
  ```

- `service <service> [command]`：对某个 serive 执行操作：

    ```console
    Available commands:
        start           Start the service
        stop            Stop the service
        restart         Restart the service
        reload          Reload configuration files (or restart if service does not implement reload)
        enable          Enable service autostart
        disable         Disable service autostart
        enabled         Check if service is started on boot
        running         Check if service is running
        status          Service status
        trace           Start with syscall trace
        info            Dump procd service info
    ```

## 附录三：LuCI 语法

以`/etc/config/dhcp`为例，其语法形式包括：

- config：表示当前节点
- option：表示节点的一个 Key-Value 元素，建议使用**单引号**包含
- list：表示列表形式的一组参数

```config
config dnsmasq
    option domainneeded '1'
    option localise_queries '1'
    option rebind_protection '1'
    option rebind_localhost '1'
    option local '/lan/'
    option domain 'lan'
    option expandhosts '1'
    option cachesize '1000'
    option readethers '1'
    option leasefile '/tmp/dhcp.leases'
    option resolvfile '/tmp/resolv.conf.d/resolv.conf.auto'
    option localservice '1'
    option ednspacket_max '1232'

config dhcp 'lan'
    option interface 'lan'
    option start '100'
    option limit '150'
    option leasetime '12h'
    option dhcpv4 'server'
    option dhcpv6 'server'
    option ra 'server'
    option ra_slaac '1'
    list ra_flags 'managed-config'
    list ra_flags 'other-config'

config dhcp 'wan'
    option interface 'wan'
    option ignore '1'

config odhcpd 'odhcpd'
    option maindhcp '0'
    option leasefile '/tmp/hosts/odhcpd'
    option leasetrigger '/usr/sbin/odhcpd-update'
    option loglevel '4'
```

---

## 参考文献

- [Linux overlayfs文件系统介绍](https://zhuanlan.zhihu.com/p/436450556)
- [OverlayFS简介](https://cn.linux-console.net/?p=10009)

### 官方文档

- 
- 
  