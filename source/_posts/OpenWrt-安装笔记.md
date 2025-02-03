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
opkg 系统源的配置文件：`/etc/opkg/distfeeds.conf`，默认指向[https://downloads.openwrt.org](https://downloads.openwrt.org/releases/23.05.3/packages/x86_64)
手工替换为阿里云的镜像源，以加快软件安装速度。

```bash
sed -i 's_downloads.openwrt.org_mirrors.aliyun.com/openwrt_' /etc/opkg/distfeeds.conf
```

处理结果如下：

```config
src/gz openwrt_core https://mirrors.aliyun.com/openwrt/releases/23.05.3/targets/x86/64/packages
src/gz openwrt_base https://mirrors.aliyun.com/openwrt/releases/23.05.3/packages/x86_64/base
src/gz openwrt_luci https://mirrors.aliyun.com/openwrt/releases/23.05.3/packages/x86_64/luci
src/gz openwrt_packages https://mirrors.aliyun.com/openwrt/releases/23.05.3/packages/x86_64/packages
src/gz openwrt_routing https://mirrors.aliyun.com/openwrt/releases/23.05.3/packages/x86_64/routing
src/gz openwrt_telephony https://mirrors.aliyun.com/openwrt/releases/23.05.3/packages/x86_64/telephony
```

> 清华源似乎有点问题，PassWall 安装时有兼容性问题

### 3. LuCI = Lua + UCI

[UCI（Unified Configuration Interface）](https://openwrt.org/zh/docs/guide-user/base-system/uci)是 Openwrt 中为实现所有系统配置的统一配置接口，只包括一个精简的核心和最基本的库，体积小、启动速度快，意在 实现 OpenWrt 整个系统的配置集中化。
LuCI 是一个使用 Lua 语言开发的 Web UI，采用 MVC 三层架构，提供 OpenWrt 的系统管理功能。
LuCI 的统一配置文件目录：`/etc/config/`，登录方式为 `http://<LAN IP>:80`。

## 二、基础系统安装

准备好基础镜像后，就可以创建一个虚拟机准备安装了。

### 1. VM 准备

建议配置 1C + 1G、默认硬盘（无所谓，后面会删除）、默认 Lan 端口（因为是旁路无需增加 Wan 端口）。

- 将基础镜像 SCP 到 node（OpenWrt默认不支持 sftp）的目录空间
- 登录 node 的 Shell，加载基础镜像磁盘（展开后约 124 MB）
    `qm importdisk <VM ID> <Img Filename> local-lvm`
- 删除 CDROM 和默认硬盘；加载镜像盘并**适当扩容**（建议 1GB）；
- 注意检查 **Boot Order**，确保从镜像硬盘启动！

现在启动 VM，如果一切正常就可以从 Console 登录了。
建议替换 opkg 的软件源为国内镜像，以加快后续安装速度

### 2. 网络配置

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

### 3. 磁盘扩容

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

#### 新建磁盘（可选）

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

这一步也可通过 LuCI 的 system - Mount Points 页面的 Mount Points 段落的 Add 按钮进行编辑

### 4. 其他配置

OpenWrt 安装启动后，许多操作就可以通过 LuCI 操作了。

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

## 三、Transmission 部署

1. 安装软件包

    ```bash
    opkg update
    opkg install luci-app-transmission transmission-web
    ```

2. 刷新 LuCI 界面，将新增出现 Service - Transmission 菜单；
    编辑如下字段，Save & Apply 保存并应用配置信息
  
   - **Enabled**：勾选。
   - Run daemon as user & Run daemon as group：修改 transmission -> root。不为权限烦恼！
   - Config file directory：默认 /tmp/transmission。加挂磁盘后再次修改！
   - Download directory：默认 /tmp/transmission/done。加挂磁盘后再次修改，这也是以后的共享目录！
   - Incomplete directory enabled：默认取消。没啥好处
   - Cache size in MB：默认 2M

3. 手工启动 transmission 服务

    ```bash
    service transmission start
    ```

现在可以打开 Transmission 管理界面：`http://<LAN IP>:9091`。软件安装已完成！
现在下载数据目录`/tmp/transmission/` 的内容如下：

```console
drwxr-xr-x    2 root     root          4096 Jan 12 02:29 blocklists
drwxr-xr-x    2 root     root          4096 Jan 12 02:29 done
drwxr-xr-x    2 root     root          4096 Jan 12 02:29 resume
-rw-r--r--    1 root     root          1355 Jan 12 02:29 settings.json
-rw-r--r--    1 root     root             0 Jan 12 02:29 stats.json
drwxr-xr-x    2 root     root          4096 Jan 12 02:29 torrents
```

### 后续调整磁盘

后续使用时需要大容量磁盘，处理步骤包括：

1. 为 VM 添加磁盘
2. 对新增磁盘建立分区、格式化文件系统、设置 fstab
3. 进入 Transmission 配置页面，将下载目录调整到磁盘挂载目录`/mnt/transmission/`
4. 重启 Transmission 服务

## 四、Samba4 部署

1. 安装软件包

    ```bash
    opkg update
    opkg luci-app-samba4
    ```

    如果是挂载其他文件系统格式的硬盘，需要安装相应的软件驱动：
    `opkg install kmod-fs-ext4 kmod-fs-exfat kmod-fs-ntfs3`
    如果挂载是通过 USB 外接硬盘，还需要安装相关 USB 驱动和模块：
    `opkg install kmod-usb3 kmod-usb-storage-uas usbutils block-mount mount-utils luci-app-hd-idle`
2. 软件包安装成功后，LuCI 新增 Services - Network Shares 菜单。
   - Interface：选择提供服务的网络端口
   - Enable extra Tuning：建议勾选
   - Enable macOS compatible shares：建议勾选
   - Shared Directories 段落：共享目录管理，按需要操作即可。建议勾选 Force Root 简化权限问题。
3. 无需手工处理，samba4 服务已经启动，现在可以发现网络共享目录了！

## 五、Frp 部署

1. 安装软件包。frpc 是客户端软件，也可以根据需要安装服务端软件 frps。

    ```bash
    opkg update
    opkg install luci-app-frpc
    ```

2. 软件包安装成功后，LuCI 新增 Services -frp client 菜单。
   - Server address：服务端域名或 IP 地址
   - Server port：服务端的管理端口号
   - Token：服务端的鉴权凭证
   - Proxy Settings 段落：代理服务设置，支持 tcp udp http https stcp xtcp 等类型
3. 无需手工启动服务，但如果服务端配置错误，frpc 服务将不可见。

## 六、Passwall 部署

本文略。注意需要通过 SCP 拷贝 ipk 文件，并在本地手工安装。

## 七、简要分析

以 Transmission 安装包为例，对 OpenWrt 技术特点做个简要分析。先说结论：

1. OpenWrt 软件安装应选择 `luci-app-xxx` 的安装包，其不仅包含了软件的基础功能，并通过 UCI 统一配置接口支持 LuCI 界面配置。
2. 应用安装后，不要按照 Linux 习惯去 /etc/xxx 目录管理配置，现在都在 /etc/config/ 目录集中管理。
3. 系统服务管理是传统的 Server 模式，不是现代的 Systemd 模式。
4. LuCI 界面的汉化语言包命名为 `luci-i18n-xxx-zh-cn`，不建议安装，原文更准确！

`luci-app-transmission` 是 Transmission 在 OpenWrt 的软件包名称，分析其构成。
一是有 2 个底层依赖包，分别是 transmission-deamon 和 libc；
二是自带 3 个文件，其实是 LuCI 定制菜单 Service-Transmission 的配置信息。

```console
root@OpenWrt:~# opkg info luci-app-transmission
Package: luci-app-transmission
Version: git-24.364.71483-75d2b84
Depends: libc, transmission-daemon                                  # 2个依赖包
Status: install user installed
Section: luci
Architecture: all
Size: 3807
Filename: luci-app-transmission_git-24.364.71483-75d2b84_all.ipk    # 实际 ipk 文件名
Description: LuCI Support for Transmission
Installed-Time: 1736645046

root@OpenWrt:~# opkg files luci-app-transmission
Package luci-app-transmission (git-24.364.71483-75d2b84) is installed on root and has the following files:
/www/luci-static/resources/view/transmission.js                     # LuCI 的定制 UI 
/usr/share/rpcd/acl.d/luci-app-transmission.json
/usr/share/luci/menu.d/luci-app-transmission.json
```

继续分析`transmission-daemon` ，这是 Transmission 提供的，与其他 Linux 系统的软件包完全一致。

```console
root@OpenWrt:~# opkg files transmission-daemon
Package transmission-daemon (4.0.6-1) is installed on root and has the following files:
/etc/seccomp/transmission-daemon.json       # 系统服务参数
/etc/init.d/transmission                    # 系统启动脚本
/etc/sysctl.d/20-transmission.conf          # 环境参数配置文件
/usr/bin/transmission-daemon                # 二进制代码
/etc/config/transmission                    # 应用参数配置文件
```

至于`libc`就没啥说的了，这是更基础的 Linux 系统提供的动态链接库。

```console
root@OpenWrt:~# opkg files libc
Package libc (1.2.4-4) is installed on root and has the following files:
/lib/ld-musl-x86_64.so.1                    # 软链接，用于动态连接库的版本控制
/lib/libc.so                                # 动态连接库
/usr/bin/ldd                                # 二进制代码，用于判断某个 binary 档案含有什么动态函式库
```

此外，Transmisson 还提供了几个可选软件包：

- transmission-web：必选。标准的下载管理 UI 界面。
- transmission-web-control：与 transmission-web 互斥。另一个下载管理 UI 界面，信息更丰富。
- transmission-cli：可选。命令行工具。
- transmission-remote：可选。远程操作组件。

---

## 附录一：UCI 统一配置接口

UCI 的统一配置文件目录：`/etc/config/`，每个系统服务都有一个相应的配置文件。

|配置文件名|服务描述|
|:-|:-:|
|/etc/config/dhcp|dnsmasq和DHCP的配置|
|/etc/config/dropbear|SSH服务的替代品|
|/etc/config/firewall|Linux系统防火墙配置|
|/etc/config/network|本机网络接口和路由配置|
|/etc/config/system|杂项与系统配置|
|/etc/config/wireless|无线接口设置和无线网络定义|
|/etc/config/fstab|挂载点及swap|
|/etc/config/luci|基础 LuCI 配置|
|/etc/config/mountd|OpenWrt 自动挂载进程(类似autofs)|
|/etc/config/samba|samba配置(Microsoft文件共享)|
|/etc/config/transmission|BitTorrent配置|

所有系统服务清单及其作用，参见[OpenWrt UCI 系统](https://openwrt.org/zh/docs/guide-user/base-system/uci)。

### 配置文件的语法

以`/etc/config/network`为例，其语法形式包括：

- config：表示当前节点（section）
- option：表示节点的一个元素（key-value），建议使用**单引号**包裹 value
- list：表示列表（list）形式的一组参数

```config
config interface 'loopback'
    option device 'lo'
    option proto 'static'
    option ipaddr '127.0.0.1'
    option netmask '255.0.0.0'

config globals 'globals'
    option ula_prefix 'fdfc:4d51:93dd::/48'

config device
    option name 'br-lan'
    option type 'bridge'
    list ports 'eth0'

config interface 'lan'
    option device 'br-lan'
    option proto 'dhcp'
```

### 配置文件的操作

uci 是 UCI 的命令行工具，提供 add set get show delete rename commit 等操作方法。
例如，磁盘扩容的几个命令行，换个角度看就是去处理 `/etc/config/fstab` 文件，其结果如下：

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

### 系统服务的启动

OpenWrt 软件包一般采用 UCI 模式管理，通常是以 `luci-app-xxx` 形式命名的软件包，其典型启动流程为：

1. 启动脚本 `/etc/init.d/samba`，或者 `serive samba <start|stop|restart|enable|disable>`
2. 启动脚本通过 UCI 分析库从 `/etc/config/samba` 获得启动参数
3. 应用启动脚本完成正常启动

service 命令的执行过程实际上就是操作`/etc/init.d/`目录下的初始化脚本，这是传承自 System V 的标准风格。
值得一提的是，比`/etc/init.d/`优先级更高的是`/etc/rc[0..6].d/` 目录，其设定了不同级别的启动脚本，这也是 System V 的典型风格。当然，还有最高优先级的`/etc/inittab`脚本。

OpenWrt 的系统服务管理命令：

- `service`：显示系统服务列表，包括服务名称、初始状态、运行状态

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

## 附录二：opkg 常用命令

opkg 环境的配置目录：`/etc/opkg.conf`
opkg 源的配置目录：`/etc/opkg/`，具体内容为：

- `distfeeds.conf` ：默认软件源的路径，指向 [https://downloads.openwrt.org](https://downloads.openwrt.org/releases/23.05.3/packages/x86_64)
- `customfeeds.conf`：用户软件源，初始为空。
- `keys/`：用于密钥管理

opkg 常用命令：

- opkg update：更新可以获取的软件包列表
- opkg upgrade：对已经安装的软件包升级
- opkg list：获取软件列表
- opkg list installed：获取已安装的软件列表
- opkg install < pkg >：安装指定的软件包
- opkg remove < pkg >：卸载已经安装的指定的软件包
- opkg info < pkg >：获取某个软件包的完整信息，例如底层依赖
- opkg files < pkg>：获取某个软件包的文件清单
  
## 附录三：瑞士军刀 busybox

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

---

## 官方文档

- [OpenWrt 历史版本清单](https://downloads.openwrt.org/releases/)
- [OpenWrt 系统镜像下载](https://downloads.openwrt.org/releases/23.05.3/targets/x86/64)
- [OpenWrt 软件源目录](https://mirror-03.infra.openwrt.org/releases/23.05.3/packages/x86_64/)
- [OpenWrt UCI 介绍](https://openwrt.org/zh/docs/guide-user/base-system/uci)
- [NTP client / NTP server](https://openwrt.org/docs/guide-user/services/ntp/client-server)

## 参考文献

- [如何在 PVE 上安装一个官方版的 Openwrt](https://www.barhe.org/archives/1234)
- [uci命令系统详解](https://blog.csdn.net/qq_35718410/article/details/53113894)
- [Linux overlayfs文件系统介绍](https://zhuanlan.zhihu.com/p/436450556)
- [OverlayFS简介](https://cn.linux-console.net/?p=10009)
- [Openwrt 作为旁路网关（不是旁路由、单臂路由）的终极设置方法](https://sspai.com/post/68511)
