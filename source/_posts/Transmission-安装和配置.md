---
title: Transmission 安装和配置
date: 2022-04-16 19:16:25
tags:
---

## 一、概述

Transmission是一个轻量级、跨平台、开源的 BitTorrent 客户端，官网地址是[https://transmissionbt.com/](https://transmissionbt.com/)，当前最新版本是3.0。
![Transmission 的官网首页](home.png)

其实现了BT协议中描述的大多数功能（除了不能创建种子以外），覆盖了Linux、MacOS、Windows等所有主流操作系统，核心组件包括：

- 1个基于MacOS的本地应用程序
- 2个基于Linux GUI的本地应用程序，分别支持 GTK 和 QT
- 1个基于Linux的后台服务，用于服务器或路由器
- 1个基于WEB的UI界面，可以支持以上所有组件

下面以Centos为例，介绍其安装和配置的基本步骤。

## 二、软件源和安装方法

最简便的方法是使用EPEL源，通过`yum search transmission`搜索，主要有以下软件包：

- `transmission-common`: 最基础软件，包含1个 Web GUI 和 4个命令行程序：`transmission-create | edit | remote | show`
- `transmission-daemon`: = `transmission-common` + 1个后台守护程序`transmission-daemon`
- `transmission-cli`: = `transmission-common` + 1个系统配置命令行程序`transmission-cli`
- `transmission-gtk`：= `transmission-common` + 基于GTK GUI的Applicaition
- `transmission-qt`：= `transmission-common` + 基于Qt GUI的Applicaition
- `transmission`: = 常用安装包名，默认采用GTK，基本等于`transmission-gtk`

因此，最简单的安装方法就是：`yum install transmission`，当然严谨一点就是:

``` bash
    yum install transmission-daemon
```

可执行程序的安装位置是`/usr/bin/`，可以发现有`transmission-daemon`等5个程序。

``` bash
[root@localhost transmission-daemon]# ls -l /usr/bin/transmission*
-rwxr-xr-x 1 root root 560880 5月  18 2020 /usr/bin/transmission-create
-rwxr-xr-x 1 root root 577152 5月  18 2020 /usr/bin/transmission-daemon
-rwxr-xr-x 1 root root 556456 5月  18 2020 /usr/bin/transmission-edit
-rwxr-xr-x 1 root root 597360 5月  18 2020 /usr/bin/transmission-remote
-rwxr-xr-x 1 root root 556464 5月  18 2020 /usr/bin/transmission-show
```

安装完成后检查`/etc/passwd`，发现添加了一个不能登录的系统用户`transmission`，HOME就是默认安装目录`/var/lib/transmission`。
HOME的目录结构如下，注意：配置文件隐藏在`$HOME/.config/transmission-daemon/`，而不是常规的`/etc/`

``` console
[root@test transmission]# tree /var/lib/transmission -a
/var/lib/transmission
├── .config/                    # 配置文件目录
│   └── transmission-daemon/ 
│       ├── blocklists/         # 各个种子文件的数据块信息
│       ├── resume/             # 各个种子文件的运行状态信息
│       ├── settings.json       # 主配置文件，json格式
│       ├── dht.dat             # 存储DHT节点信息
│       └── torrents/           # 种子文件的存储目录
├── Downloads/                  # 下载数据文件存储目录
└── .pki/
    └── nssdb/
```

## 三. 启动方式

`transmission-daemon`支持`systemd`启动方式。

``` bash
systemctl start transmission-daemon --now
systemctl status transmission-daemon
```

如果后台服务正常启动，可以看到如下启动信息：

``` console
[root@test transmission]# systemctl status transmission-daemon
● transmission-daemon.service - Transmission BitTorrent Daemon
   Loaded: loaded (/usr/lib/systemd/system/transmission-daemon.service; disabled; vendor preset: disabled)
   Active: active (running) since 六 2022-04-16 22:10:31 CST; 34min ago
 Main PID: 12051 (transmission-da)
   Status: "Idle."
   CGroup: /system.slice/transmission-daemon.service
           └─12051 /usr/bin/transmission-daemon -f --log-error

4月 16 22:10:31 test.caogo.local systemd[1]: Starting Transmission BitTorrent Daemon...
4月 16 22:10:31 test.caogo.local systemd[1]: Started Transmission BitTorrent Daemon.
```

### 常见问题1：UDP缓冲区不足导致Daemon启动失败

如果启动失败，并产生如下信息，通常是操作系统的UDP网络缓冲区不足

``` txt
UDP Failed to set receive buffer: requested 4194304, got 425984 (tr-udp.c:84)
UDP Failed to set send buffer: requested 1048576, got 425984 (tr-udp.c:95)
```

解决办法是通过`sysctl -p`命令来检查，并修改位于`/etc/sysctl.conf`的内核参数

```console
echo 'net.core.rmem_max = 16777216' >> /etc/sysctl.conf
echo 'net.core.wmem_max = 4194304' >> /etc/sysctl.conf
sysctl -p
```

## 四. 使用方式

后台服务启动成功后，可以看到占用tcp/udp 51413作为BT通信端口，占用tcp 9091端口用于访问Web UI。

``` console
[root@test lib]# netstat -tunpl |grep transmission
tcp        0      0 0.0.0.0:51413           0.0.0.0:*               LISTEN      11846/transmission- 
tcp        0      0 0.0.0.0:9091            0.0.0.0:*               LISTEN      11846/transmission- 
tcp6       0      0 :::51413                :::*                    LISTEN      11846/transmission- 
udp        0      0 0.0.0.0:51413           0.0.0.0:*                           11846/transmission- 
```

此时，通过浏览器打开`http://<IP地址>:9091`，就可以看到Web UI的局面了。

![Web UI](web.png)

### 常见问题2: Web UI界面无法打开

如果浏览器提示“403: Forbidden”错误信息，并提示“Unauthorized IP Addres”，通常是因为默认只允许来自本机`127.0.0.1`的白名单访问。
解决方法是：

1. 停止守护进程，否则修改无法存盘；

    ``` bash
    systemctl stop transmission-daemon
    ```

2. 编辑配置文件，修改`rpc-whitelist-enabled`为`false`

    ``` bash
    vi /var/lib/transmission/.config/transmission-daemon/settings.json
    ```

3. 重新启动守护进程，并打开浏览器正常访问。

    ``` bash
    systemctl start transmission-daemon
    ```

## 五、跨平台的安装说明

### MacOS

MacOS下的Transmission是一个DMG安装包，已经包含了GUI界面。
注意，其配置文件目录是：

- 每个种子文件的状态信息位于：`~/Library/Application Support/Transmission`
- 全局配置文件位于：`~/Library/Preferences/org.m0k.transmission.plist`

### 嵌入式设备

群晖NAS、西部数据NAS、D-Link路由器等各种嵌入式设备，由于其都是基于Linux核心，因此也可以安装Transmission。
以西部数据NAS为例，MyBookLive、MyCloud等设备在技术上都可支持，但由于不属于官方项目，因此版本升级时经常被限制。
具体安装方法参见[http://mybookworld.wikidot.com/optware](http://mybookworld.wikidot.com/optware)

---

## 附录一：全量配置参数

以Centos为例，其配置文件位于`/var/lib/transmission/.config/transmission-daemon/settings.json`：

``` bash
"alt-speed-up": 500, #计划时段上传限速值
"alt-speed-down": 500, #计划时段下载限速值
"alt-speed-enabled": false,
"alt-speed-time-begin": 540,
"alt-speed-time-day": 127,
"alt-speed-time-enabled": true, #启用计划工作，为false时，以上计划配置则不生效
"alt-speed-time-end": 420, #计划结束时间，为零点到开始时间的分钟数，比如7:00就是7*60=420。另外，该时间是用的GMT时间，即北京时间-8小时。比如你计划北京时间7点30分开始，这个数字应该是（7-8+24）*60+30=1410
"bind-address-ipv4": "0.0.0.0",
"bind-address-ipv6": "::",
"blocklist-enabled": true,
"blocklist-updates-enabled": false,
"blocklist-url": "http://www.example.com/blocklist",
"cache-size-mb": 4, #缓存大小，以MB为单位，建议设大一些，避免频繁读写硬盘而伤硬盘，建议设为内存大小的1/6～1/4
"compact-view": false,
"dht-enabled": false, #关闭DHT（不通过tracker寻找节点）功能，不少PT站的要求，但BT下载设置为true会使得下载更好
"download-dir": "/home/yys/Downloads", #下载的内容存放的目录
"download-queue-enabled": true,
"download-queue-size": 5,
"encryption": 1, #0=不加密，1=优先加密，2=必须加密
"idle-seeding-limit": 30,
"idle-seeding-limit-enabled": false,
"incomplete-dir": "/home/yys/Downloads",
"incomplete-dir-enabled": false,
"inhibit-desktop-hibernation": true,
"lpd-enabled": false, #禁用LDP（本地节点发现，用于在本地网络寻找节点）,不少PT站的要求
"main-window-height": 500,
"main-window-is-maximized": 0,
"main-window-width": 615,
"main-window-x": 337,
"main-window-y": 211,
"message-level": 2,
"open-dialog-dir": "/home/yys/\u684c\u9762",
"peer-congestion-algorithm": "",
"peer-limit-global": 240, #全局连接数
"peer-limit-per-torrent": 60, #每个种子最多的连接数
"peer-port": 51413, #预设的port口
"peer-port-random-high": 65535,
"peer-port-random-low": 49152,
"peer-port-random-on-start": false, #不建议改为true
"peer-socket-tos": "default",
"pex-enabled": false, #禁用PEX（节点交换，用于同已与您相连接的节点交换节点名单）,不少PT站的要求
"port-forwarding-enabled": true,
"preallocation": 1, #预分配文件磁盘空间，0=关闭，1=快速，2=完全。建议取1开启该功能，防止下载大半了才发现磁盘不够。取2时，可以减少磁盘碎片，但速度较慢。
"prefetch-enabled": 1,
"queue-stalled-enabled": true,
"queue-stalled-minutes": 30,
"ratio-limit": 2,
"ratio-limit-enabled": false,
"rename-partial-files": true, #在未完成的文件名后添加后缀.part,false=禁用
"rpc-authentication-required": true,
"rpc-bind-address": "0.0.0.0",
"rpc-enabled": true,
"rpc-password": "{c8c083168db9fff40b5136b6d0f3f4a864110a78\/oH51JaE", #web-ui的密码，可直接修改，重新运行或者reload服务的时候会自动被加密
"rpc-port": 9091, #默认web-ui的port口，可自行更改
"rpc-url": "/transmission/",
"rpc-username": "transmission", #默认登入名称
"rpc-whitelist": "127.0.0.1",
"rpc-whitelist-enabled": true, #如果你要让其他网段连入，请设false
"scrape-paused-torrents-enabled": true,
"script-torrent-done-enabled": false,
"script-torrent-done-filename": "/home/yys",
"seed-queue-enabled": false,
"seed-queue-size": 10,
"show-backup-trackers": true,
"show-extra-peer-details": false,
"show-filterbar": true,
"show-notification-area-icon": false,
"show-options-window": true,
"show-statusbar": true,
"show-toolbar": true,
"show-tracker-scrapes": true,
"sort-mode": "sort-by-age",
"sort-reversed": false,
"speed-limit-down": 300, #平时的下载限速
"speed-limit-down-enabled": true, #启用平时下载限速
"speed-limit-up": 30, #平时上传限速
"speed-limit-up-enabled": true, #启用平时上传限速
"start-added-torrents": false,
"statusbar-stats": "total-ratio",
"torrent-added-notification-enabled": true,
"torrent-complete-notification-enabled": true,
"torrent-complete-sound-enabled": true,
"trash-can-enabled": true,
"trash-original-torrent-files": false,
"umask": 18,
"upload-slots-per-torrent": 14
"utp-enabled": true, #启用μTP协议
"watch-dir": "/home/yys/\u4e0b\u8f7d",
"watch-dir-enabled": false
```

## 附录二：关于Qt 和 GTK

### Qt

1991年，Haavard Nord和Eirik Chambe-Eng开发了“Qt”，该工具包名为Qt是因为字母Q在Haavard的Emacs字体特别漂亮，而“t”代表“toolkit”，灵感来自Xt，X toolkit。
后来，两人成立了Trolltech公司（中文名是“奇趣科技”），2008年被NOKIA公司收购，以增强该公司在跨平台软件研发方面的实力，更名Qt Software并宣布开放Qt源代码，
2012年8月9日，Digia宣布已完成对诺基亚Qt业务及软件技术的全面收购，并计划将Qt应用到Android、iOS及Windows 8平台上。

Qt的图形用户界面的基础是QWidget。Qt中所有类型的GUI组件如按钮、标签、工具栏等都派生自QWidget，而QWidget本身则为QObject的子类。Widget负责接收鼠标，键盘和来自窗口系统的其他事件，并描绘了自身显示在屏幕上。每一个GUI组件都是一个widget，widget还可以作为容器，在其内包含其他Widget。
使用Qt开发的软件，相同的代码可以在任何支持的平台上编译与执行，而不需要修改源代码。会自动依平台的不同，表现平台特有的图形界面风格。
Qt开放源代码，并提供LGPL和GPL的自由软件用户协议，可以免费使用，但商业版需收取授权费。

KDE Plasma Workspaces就是基于Qt开发的Linux GUI，此外Symbain、MeeGo等手机厂商也采用Qt框架，但现在是Android的天下？

### GTK

GTK（原名GTK+）最初是GIMP的专用开发库（GIMP Toolkit），后来发展为类Unix系统下开发图形界面的应用程序的主流开发工具之一。
GTK是自由软件，并且是GNU计划的一部分。自2019年2月6日起，GTK+改名为GTK。
GTK使用C语言开发，但使用了面向对象技术，也提供了C++（gtkmm）、Perl、Ruby、Java和Python（PyGTK）绑定，其他的绑定有Ada、D、Haskell、PHP和所有的.NET编程语言。

GNOME是以GTK为基础，就是说为GNOME编写的程序使用GTK做为其工具箱，Firefox也是基于GTK开发的。

---

## 参考文献

- [Transmission 的官网 - transmissionbt.com](https://transmissionbt.com/)
- [Transmission 的源码](https://github.com/transmission/transmission)
- [Qt 的 Wiki](https://zh.wikipedia.org/wiki/Qt)
- [GTK 的 Wiki](https://zh.wikipedia.org/wiki/GTK)
- [Transmission 的安装与配置 - Ubuntu发行版](https://blog.uuz.moe/2017/02/install_transmission/)
- [Transmission 的安装与配置 - Archlinux发行版](https://wiki.archlinux.org/title/Transmission_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E6%8E%A5%E6%94%B6/%E5%8F%91%E9%80%81%E7%BC%93%E5%86%B2%E5%8C%BA%E8%AE%BE%E7%BD%AE%E5%A4%B1%E8%B4%A5)
- [UDP缓冲区不足导致daemon启动失败的解决方案](http://ronhks.hu/2018/12/30/transmission-network-problem/)
