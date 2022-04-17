---
title: Transmission 安装和配置
date: 2022-04-16 19:16:25
tags:
---

## 一、概述

Transmission是一个轻量级、跨平台、开源的 BitTorrent 客户端，官网地址是[https://transmissionbt.com/](https://transmissionbt.com/)，当前最新版本是3.0。
![Transmission 的官网首页](home.png)

其实现了BT协议的几乎全部功能，覆盖了Linux、MacOS、Windows等所有主流操作系统，核心组件包括：

- 1个基于MacOS的Native Application
- 2个基于Linux的Native Application，分别支持 GTK GUI和 QT GUI
- 1个基于Linux的守护服务，用于服务器或路由器的后台服务
- 1个WEB UI前端界面，通过Json RPC接口服务提供浏览器访问

![Transmission技术架构](arch.gif)

## 二、Centos的安装配置方法

下面以Centos为例，介绍其安装和配置的基本步骤。

### 1. 软件包安装

最简便的方法是使用EPEL源，简单粗暴就是：`yum install transmission`，当然严谨一点就是:

``` bash
yum install transmission-daemon
```

### 2. 系统自启动

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

### 3. 运行监控

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

## 三、Transmission软件包的构成分析

最简便的方法是使用EPEL源，通过`yum search transmission`搜索，主要有以下软件包：

- `transmission-common`: 核心组件，包含1个 Web GUI 和 4个命令行程序：`transmission-create | edit | remote | show`
- `transmission-daemon`: = `transmission-common` + 1个后台守护程序`transmission-daemon`
- `transmission-cli`: = `transmission-common` + 1个系统配置命令行程序`transmission-cli`
- `transmission-gtk`：= `transmission-common` + 基于GTK GUI的Applicaition
- `transmission-qt`：= `transmission-common` + 基于Qt GUI的Applicaition
- `transmission`: = 常用安装包名，默认采用GTK，基本等于`transmission-gtk`

通过`yum info transmission-common`可以查看该软件包的基本信息。
通过`repoquery -ql transmission-common.x86_64`，可以查看该软件包的全部文件信息。

``` console
[root@test transmission]# yum info transmission-common
已加载插件：fastestmirror
Loading mirror speeds from cached hostfile
已安装的软件包
名称    ：transmission-common
架构    ：x86_64
版本    ：2.94
发布    ：9.el7
大小    ：3.0 M
源    ：installed
来自源：epel
简介    ： Transmission common files
网址    ：http://www.transmissionbt.com
协议    ： MIT and GPLv2
描述    ： Common files for Transmission BitTorrent client sub-packages. It includes
         : the web user interface, icons and transmission-remote, transmission-create,
         : transmission-edit, transmission-show utilities.

[root@test transmission]# repoquery -ql transmission-daemon.x86_64
/usr/bin/transmission-daemon
/usr/lib/systemd/system/transmission-daemon.service
/usr/share/man/man1/transmission-daemon.1.gz
/var/lib/transmission
```

由此可以分析得出`transmission-daemon`安装过程的主要步骤包括：

### 1. 创建默认用户

安装完成后检查`/etc/passwd`，发现添加了一个不能登录的系统用户`transmission`。

``` txt
transmission:x:998:995:transmission daemon account:/var/lib/transmission:/sbin/nologin
```

HOME就是软件包的默认安装目录`/var/lib/transmission`。

### 2. 拷贝可执行程序

可执行程序的安装位置是`/usr/bin/`，可以发现有`transmission-daemon`等5个程序。

``` bash
[root@localhost transmission-daemon]# ls -l /usr/bin/transmission*
-rwxr-xr-x 1 root root 560880 5月  18 2020 /usr/bin/transmission-create
-rwxr-xr-x 1 root root 577152 5月  18 2020 /usr/bin/transmission-daemon
-rwxr-xr-x 1 root root 556456 5月  18 2020 /usr/bin/transmission-edit
-rwxr-xr-x 1 root root 597360 5月  18 2020 /usr/bin/transmission-remote
-rwxr-xr-x 1 root root 556464 5月  18 2020 /usr/bin/transmission-show
```

### 3. 拷贝软件文档

yum安装软件包时，通常在以下目录补充该软件的文档信息：

- /usr/share/doc/：本软件使用说明书
- /usr/share/icons/：本软件使用的各种尺寸的图标
- /usr/share/licenses/：本软件的许可证文件
- /usr/share/man/：man命令的帮助信息
- /usr/share/pixmaps/：本软件Logo的位图

### 4. 拷贝Web UI

Transmission的Web UI的入口文件存储在`/usr/share/transmission/web/index.html`，其结构与普通网站一致：

``` console
web
├── images
├── index.html
├── javascript
├── LICENSE
└── style
    ├── jqueryui
    └── transmission
```

### 5. 设置系统启动服务

采用systemd的启动方式，服务配置文件是`/usr/lib/systemd/system/transmission-daemon.service`

``` config
[Unit]
Description=Transmission BitTorrent Daemon
After=network.target

[Service]
User=transmission
Type=notify
ExecStart=/usr/bin/transmission-daemon -f --log-error
ExecReload=/bin/kill -s HUP $MAINPID

[Install]
WantedBy=multi-user.target
```

### 6. 设置软件包的数据目录

Transmission-daemon的配置文件隐藏在`$HOME/.config/transmission-daemon/`，而不是常规的`/etc/`
HOME的目录结构如下：

``` console
[root@test transmission]# tree /var/lib/transmission -a
/var/lib/transmission
├── .config/                    # 配置文件目录
│   └── transmission-daemon/ 
│       ├── blocklists/         # 各个种子文件的数据块信息
│       ├── resume/             # 各个种子文件的运行状态
│       ├── settings.json       # 主配置文件，json格式
│       ├── stats.json          # 当前系统运行状态，json格式
│       ├── dht.dat             # 存储DHT节点信息
│       └── torrents/           # 种子文件的存储目录
├── Downloads/                  # 下载数据文件的存储目录
└── .pki/
    └── nssdb/
```

## 四、Transmission跨平台技术方案分析

`Transmission core`作为核心引擎，负责实现BT协议的全部功能，其底层依赖两个外部库函数：

- `libevent`：一个异步事件处理软件库。libevent提供了一组应用程序编程接口（API），允许开发者为事件注册回调函数，用来取代网络服务器所使用的事件循环检查框架。
- `libnatpmp`：一个NAT-PMP协议库。NAT-PMP（Network Address Translation Port Mapping Protocol）协议通过端口映射的方式获取外网主机的IP地址。

其对外提供服务主要通过以下接口实现：

- `JSON RPC Server`：提供网络远程调用接口，以JSON格式
- `Web App Server`：基于通用Web Server，以Web形式通过网络App应用服务接口
- `Lib Transmission`：提供C语言的库函数接口，仅能用于Native Application

![APP结构](products.gif)
以上各软件包的技术依赖关系参见[Transmission技术架构](https://github.com/transmission/transmission/blob/main/docs/Transmission-Architecture.md)

### 1. MacOS

Transmission以Native Application的方式发布，形态是一个DMG安装包。
开发了一个基于GTK GUI的Native GUI，直接采用Transmission core提供的Lib库函数。

> Mac OS平台的配置文件目录存在差异，每个种子文件的状态信息位于：`~/Library/Application Support/Transmission`，而全局配置文件位于：`~/Library/Preferences/org.m0k.transmission.plist`

### 2. Windows

Transmission以Native Application的方式发布，形态是一个MSI安装包，分为32位和64位两个版本。
开发了一个基于Qt GUI的Native GUI，注意其采用的是JSON RPC接口服务，需要网络组件支持。

### 3. Linux桌面版

Linux桌面系统的市场份额很少，但是很庞杂：

- KDE：排名第一，基于Qt GUI开发
- GNOME：简单速度快，红帽等Linux发行版常用，基于GPK GUI开发
  此外，GNOME还有多个不同版本的变种，包括Unity、MATE、Cinnamon等

> Qt GUI和GPK GUI的配置文件都存储在：`$HOME/.config/transmission/`

### 4. Linux服务器

安装`transmission-daemon`提供后台服务，再通过Web UI提供管理界面是最直接的方案。
本机也可以通过`transmission-cli`提供一个字符界面的管理工具，但是没有人有兴趣使用如此简陋的工具。

> transmission-cli的配置文件存储在：`$HOME/.config/transmission-cli/`

### 5. 嵌入式设备

群晖NAS、西部数据NAS、D-Link路由器等嵌入式设备都是基于Linux核心，因此也可以安装Transmission。
以西部数据NAS为例，MyBookLive、MyCloud在技术上都可支持，但由于不属于官方项目，因此版本升级时经常被限制。
具体安装方法参见[http://mybookworld.wikidot.com/optware](http://mybookworld.wikidot.com/optware)

## 五、常见问题的解决方案

### 1：UDP缓冲区不足导致Daemon启动失败

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

### 2: Web UI界面无法打开

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

KDE Plasma Workspaces就是基于Qt开发的Linux GUI，此外Symbain、MeeGo等手机厂商也采用Qt框架，但现在已经是Android的天下了！！！

### GTK

GTK（原名GTK+）最初是GIMP的专用开发库（GIMP Toolkit），后来发展为类Unix系统下开发图形界面的应用程序的主流开发工具之一。
GTK是自由软件，并且是GNU计划的一部分。自2019年2月6日起，GTK+改名为GTK。
GTK使用C语言开发，但使用了面向对象技术，也提供了C++（gtkmm）、Perl、Ruby、Java和Python（PyGTK）绑定，其他的绑定有Ada、D、Haskell、PHP和所有的.NET编程语言。

GNOME是以GTK为基础，就是说为GNOME编写的程序使用GTK做为其工具箱，Firefox也是基于GTK开发的。

---

## 参考文献

- [Transmission 的官网 - transmissionbt.com](https://transmissionbt.com/)
- [Transmission 的源码](https://github.com/transmission/transmission)
- [settings.json 的参数设置](https://github.com/transmission/transmission/blob/main/docs/Editing-Configuration-Files.md))
- [Qt 的 Wiki](https://zh.wikipedia.org/wiki/Qt)
- [GTK 的 Wiki](https://zh.wikipedia.org/wiki/GTK)
- [Transmission 的安装与配置 - Ubuntu发行版](https://blog.uuz.moe/2017/02/install_transmission/)
- [Transmission 的安装与配置 - Archlinux发行版](https://wiki.archlinux.org/title/Transmission_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E6%8E%A5%E6%94%B6/%E5%8F%91%E9%80%81%E7%BC%93%E5%86%B2%E5%8C%BA%E8%AE%BE%E7%BD%AE%E5%A4%B1%E8%B4%A5)
- [UDP缓冲区不足导致daemon启动失败的解决方案](http://ronhks.hu/2018/12/30/transmission-network-problem/)
