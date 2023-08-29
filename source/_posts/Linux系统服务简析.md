---
title: Linux系统服务简析
date: 2023-08-29 22:06:52
tags:
---

Systemd 是一系列工具的集合，其作用也远远不仅是启动操作系统，它还接管了后台服务、结束、状态查询，以及日志归档、设备管理、电源管理、定时任务等许多职责，并支持通过特定事件（如插入特定 USB 设备）和特定端口数据触发的 On-demand（按需）任务。

Systemd 的后台服务还有一个特殊的身份——它是系统中 PID 值为 1 的进程。

Systemd 是 Linux 系统中一个重要的系统和服务管理器，最早是为了代替传统的 SysV 初始化系统（init）而开发的，相较于传统 init，systemd 具有许多优势。例如支持并行启动，可同时启动多个服务，提高系统启动速度；引入了单一进程（PID 1）和 cgroups 技术，可以更好地管理系统和服务进程。目前，许多主流 Linux 发行版都采用了 systemd 作为其默认的初始化系统，包括 Ubuntu、Debian、Fedora、CentOS、Arch Linux 等。

总的来说，使用 systemd 可以更加简单灵活地管理各种系统服务，它提供了统一的命令行工具和配置文件格式，使得对系统和服务的管理更加一致和简化。用户可以通过 systemctl 命令来控制 systemd 系统和管理服务。

![系统架构](arch.png)

## 一、概述

```sh
# 列出所有定义 systemd 服务
systemctl list-unit-files

# 列出启动失败的 systemd 服务
systemctl list-units --state failed

# 检查某个系统服务是否失败
systemctl is-failed [unit]

# 检查某个服务的运行状态
systemctl status [unit]
```

systemd有自己的日志系统，称为journald。它替换了sysVinit中的syslogd。

```bash
# 检查 systemd 服务启动的全量日志
journalctl

# 检查某个服务的启动日志
journalctl -u [unit]
```

## 二、基础的系统服务

### kdump.service - Crash recovery kernel arming

### lm_sensors.service - Hardware Monitoring Sensors

### rngd.service - Hardware RNG Entropy Gatherer Daemon

rngd 服务，是 rng-tools 软件包的部分，能够使用环境噪声和硬件随机数生成器来生成熵。 这个服务检查是否有充分随机的随机性源来提供数据并存储它到内核的随机数熵池（random-number entropy pool）。 这个随机数是通过 /dev/random 和 /dev/urandom 字符设备来生成的。

### sshd.service - OpenSSH server daemon

### postfix.service - Postfix Mail Transport Agent

基于SMTP协议的Postfix服务程序来提供发送邮件的服务功能

### rsyslog.service - System Logging Service

即：rocket-fast system for log，它提供了高性能，高安全功能和模块化设计，用于操作系统收集各种日志信息

### tuned.service - Dynamic System Tuning Daemon

监视系统组件使用情况收集的信息动态，并自动调整Linux服务器性能

### NetworkManager.service - Network Manager

### firewalld.service - firewalld - dynamic firewall daemon

### crond.service - Command Scheduler

### dbus.service - D-Bus System Message Bus

D-Bus最初为Linux而开发的“进程之间通信IPC”和“远程控制RPC”，用一个统一的协议取代当时的“进程通信”。D-Bus也被设计成允许系统级进程（例如打印机、硬件驱动程序服务）和普通进程之间的通信。
平时的通信都是采用文本格式，如往某个socket中写入“hellow”,这样传输的时候需要将文本序列化成二进制再传输，但D-Bus由于采用二进制消息传递，所以非常快、开销小，特别适合同一台主机通信。

### qemu-guest-agent.service - QEMU Guest Agent

### systemd-logind.service - Login Service

### polkit.service - Authorization Manager

PolKit（以前称为 PolicyKit）是一个应用程序框架，充当非特权用户会话与特权系统环境之间的协商者。每当用户会话中的某个进程尝试在系统环境中执行操作时，系统就会查询 PolKit。根据配置（在所谓的“策略”中指定）的不同，回答可能为“是”、“否”或“需要身份验证”。与 sudo 等传统的特权授权程序不同，PolKit 不会向整个会话授予 root 权限，而只向相关的操作授予该权限。

### auditd.service - Security Auditing Service

auditd是Linux审计系统的用户空间组件。它负责把审计记录写到磁盘上。查看日志是通过ausearch或aureport工具完成的。配置审计系统或加载规则是通过auditctl工具完成的。在启动过程中，/etc/audit/audit.rules中的规则由auditctl读取并加载到内核。另外，还有一个augenrules程序，它读取位于/etc/audit/rules.d/中的规则，并将其编译成audit.rules文件。审计守护程序本身有一些配置选项，管理员可能希望对其进行自定义。它们可以在 auditd.conf 文件中找到。

### systemd-udevd.service - udev Kernel Device Manager

### systemd-journald.service - Journal Service

默认情况下，systemd 会自动创建 slice、scope 和 service unit 的层级(slice、scope 和 service 都是 systemd 的 unit 类型，参考《初识 systemd》)，来为 cgroup 树提供统一的层级结构。

系统中运行的所有进程，都是 systemd init 进程的子进程。在资源管控方面，systemd 提供了三种 unit 类型：

- service： 一个或一组进程，由 systemd 依据 unit 配置文件启动。service 对指定进程进行封装，这样进程可以作为一个整体被启动或终止。
- scope：一组外部创建的进程。由进程通过 fork() 函数启动和终止、之后被 systemd 在运行时注册的进程，scope 会将其封装。例如：用户会话、 容器和虚拟机被认为是 scope。
- slice： 一组按层级排列的 unit。slice 并不包含进程，但会组建一个层级，并将 scope 和 service 都放置其中。真正的进程包含在 scope 或 service 中。在这一被划分层级的树中，每一个 slice 单位的名字对应通向层级中一个位置的路径。

默认情况下，系统会创建四种 slice：

- .slice：根 slice
- system.slice：所有系统 service 的默认位置
- user.slice：所有用户会话的默认位置
- machine.slice：所有虚拟机和 Linux 容器的默认位置

```console
[root@MiWiFi-RA70-srv ~]# systemctl status
● MiWiFi-RA70-srv
    State: degraded                         # 降级状态，原因是有一个服务失败
     Jobs: 0 queued
   Failed: 1 units                          # 检查发现 kdump 服务启动失败
    Since: 二 2023-08-29 22:04:01 CST; 1min 11s ago
   CGroup: /
           ├─1 /usr/lib/systemd/systemd --switched-root --system --deserialize 22
           ├─user.slice
           │ └─user-0.slice
           │   └─session-1.scope
           │     ├─1074 sshd: root@pts/0    
           │     ├─1078 -bash
           │     └─1104 systemctl status    # 这就是本命令行
           └─system.slice
             ├─rsyslog.service              # 
             │ └─814 /usr/sbin/rsyslogd -n
             ├─tuned.service
             │ └─813 /usr/bin/python2 -Es /usr/sbin/tuned -l -P
             ├─postfix.service
             │ ├─1054 /usr/libexec/postfix/master -w
             │ ├─1055 pickup -l -t unix -u
             │ └─1056 qmgr -l -t unix -u
             ├─sshd.service
             │ └─811 /usr/sbin/sshd -D
             ├─NetworkManager.service
             │ ├─496 /usr/sbin/NetworkManager --no-daemon
             │ ├─621 /sbin/dhclient -d -q -sf /usr/libexec/nm-dhcp-helper -pf /var/run/dhclient-eth0.pid -lf /var/lib/NetworkManager/dhclient-5fb06bd0-0bb0-7ffb-45f1-d6edd65f3e03-eth0.lease -cf /var/lib/NetworkManager/dhclient-eth0.conf eth0
             │ └─817 /sbin/dhclient -d -q -6 -N -sf /usr/libexec/nm-dhcp-helper -pf /var/run/dhclient6-eth0.pid -lf /var/lib/NetworkManager/dhclient6-5fb06bd0-0bb0-7ffb-45f1-d6edd65f3e03-eth0.lease -cf /var/lib/NetworkManager/dhclient6-eth0.conf eth0
             ├─crond.service
             │ └─488 /usr/sbin/crond -n
             ├─firewalld.service
             │ └─483 /usr/bin/python2 -Es /usr/sbin/firewalld --nofork --nopid
             ├─systemd-logind.service
             │ └─479 /usr/lib/systemd/systemd-logind
             ├─dbus.service
             │ └─469 /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation
             ├─polkit.service
             │ └─468 /usr/lib/polkit-1/polkitd --no-debug
             ├─qemu-guest-agent.service
             │ └─467 /usr/bin/qemu-ga --method=virtio-serial --path=/dev/virtio-ports/org.qemu.guest_agent.0 --blacklist=guest-file-open,guest-file-close,guest-file-read,guest-file-write,guest-file-seek,guest-file-flush,guest-exec,guest-exec-status -F/etc/qemu-ga/fsfreeze-hook
             ├─auditd.service
             │ └─444 /sbin/auditd
             ├─systemd-udevd.service
             │ └─381 /usr/lib/systemd/systemd-udevd
             ├─system-getty.slice
             │ └─getty@tty1.service
             │   └─494 /sbin/agetty --noclear tty1 linux
             └─systemd-journald.service
               └─353 /usr/lib/systemd/systemd-journald
```

CentOS7的服务systemctl脚本存放在:/usr/lib/systemd/,有系统（system）和用户（user）之分,需要开机不登陆就能运行的程序，存在系统服务里，即：/usr/lib/systemd/system目录下.
CentOS7的每一个服务以.service结尾，一般会分为3部分：[Unit]、[Service]和[Install] 

[Unit]部分主要是对这个服务的说明，内容包括Description和After，Description 用于描述服务，After用于描述服务类别

[Service]部分是服务的关键，是服务的一些具体运行参数的设置.

- Type=forking是后台运行的形式，
- User=users是设置服务运行的用户,
- Group=users是设置服务运行的用户组,
- PIDFile为存放PID的文件路径，
- ExecStart为服务的具体运行命令,
- ExecReload为重启命令，
- ExecStop为停止命令，
- PrivateTmp=True表示给服务分配独立的临时空间

注意：[Service]部分的启动、重启、停止命令全部要求使用绝对路径，使用相对路径则会报错！

[Install]部分是服务安装的相关设置，可设置为多用户的


```bash
UNIT                                                                                        LOAD   ACTIVE SUB       DESCRIPTION
proc-sys-fs-binfmt_misc.automount                                                           loaded active waiting   Arbitrary Executable File Formats File System Automount Point
sys-devices-pci0000:00-0000:00:01.1-ata2-host1-target1:0:0-1:0:0:0-block-sr0.device         loaded active plugged   QEMU_DVD-ROM CentOS_7_x86_64
sys-devices-pci0000:00-0000:00:05.0-virtio1-host2-target2:0:0-2:0:0:0-block-sda-sda1.device loaded active plugged   QEMU_HARDDISK 1
sys-devices-pci0000:00-0000:00:05.0-virtio1-host2-target2:0:0-2:0:0:0-block-sda.device      loaded active plugged   QEMU_HARDDISK
sys-devices-pci0000:00-0000:00:08.0-virtio2-virtio\x2dports-vport2p1.device                 loaded active plugged   /sys/devices/pci0000:00/0000:00:08.0/virtio2/virtio-ports/vport2p1
sys-devices-pci0000:00-0000:00:12.0-virtio3-net-eth0.device                                 loaded active plugged   Virtio network device
sys-devices-platform-serial8250-tty-ttyS0.device                                            loaded active plugged   /sys/devices/platform/serial8250/tty/ttyS0
sys-devices-platform-serial8250-tty-ttyS1.device                                            loaded active plugged   /sys/devices/platform/serial8250/tty/ttyS1
sys-devices-platform-serial8250-tty-ttyS2.device                                            loaded active plugged   /sys/devices/platform/serial8250/tty/ttyS2
sys-devices-platform-serial8250-tty-ttyS3.device                                            loaded active plugged   /sys/devices/platform/serial8250/tty/ttyS3
sys-module-configfs.device                                                                  loaded active plugged   /sys/module/configfs
sys-subsystem-net-devices-eth0.device                                                       loaded active plugged   Virtio network device
-.mount                                                                                     loaded active mounted   /
dev-hugepages.mount                                                                         loaded active mounted   Huge Pages File System
dev-mqueue.mount                                                                            loaded active mounted   POSIX Message Queue File System
run-user-0.mount                                                                            loaded active mounted   /run/user/0
sys-kernel-config.mount                                                                     loaded active mounted   Configuration File System
sys-kernel-debug.mount                                                                      loaded active mounted   Debug File System
systemd-ask-password-plymouth.path                                                          loaded active waiting   Forward Password Requests to Plymouth Directory Watch
systemd-ask-password-wall.path                                                              loaded active waiting   Forward Password Requests to Wall Directory Watch
session-1.scope                                                                             loaded active running   Session 1 of user root
auditd.service                                                                              loaded active running   Security Auditing Service
crond.service                                                                               loaded active running   Command Scheduler
dbus.service                                                                                loaded active running   D-Bus System Message Bus
firewalld.service                                                                           loaded active running   firewalld - dynamic firewall daemon
getty@tty1.service                                                                          loaded active running   Getty on tty1
kmod-static-nodes.service                                                                   loaded active exited    Create list of required static device nodes for the current kernel
network.service                                                                             loaded active exited    LSB: Bring up/down networking
NetworkManager-wait-online.service                                                          loaded active exited    Network Manager Wait Online
NetworkManager.service                                                                      loaded active running   Network Manager
polkit.service                                                                              loaded active running   Authorization Manager
postfix.service                                                                             loaded active running   Postfix Mail Transport Agent
qemu-guest-agent.service                                                                    loaded active running   QEMU Guest Agent
rhel-dmesg.service                                                                          loaded active exited    Dump dmesg to /var/log/dmesg
rhel-domainname.service                                                                     loaded active exited    Read and set NIS domainname from /etc/sysconfig/network
rhel-import-state.service                                                                   loaded active exited    Import network configuration from initramfs
rhel-readonly.service                                                                       loaded active exited    Configure read-only root support
rsyslog.service                                                                             loaded active running   System Logging Service
sshd.service                                                                                loaded active running   OpenSSH server daemon
systemd-journal-flush.service                                                               loaded active exited    Flush Journal to Persistent Storage
systemd-journald.service                                                                    loaded active running   Journal Service
systemd-logind.service                                                                      loaded active running   Login Service
systemd-random-seed.service                                                                 loaded active exited    Load/Save Random Seed
systemd-remount-fs.service                                                                  loaded active exited    Remount Root and Kernel File Systems
systemd-sysctl.service                                                                      loaded active exited    Apply Kernel Variables
systemd-tmpfiles-setup-dev.service                                                          loaded active exited    Create Static Device Nodes in /dev
systemd-tmpfiles-setup.service                                                              loaded active exited    Create Volatile Files and Directories
systemd-udev-trigger.service                                                                loaded active exited    udev Coldplug all Devices
systemd-udevd.service                                                                       loaded active running   udev Kernel Device Manager
systemd-update-utmp.service                                                                 loaded active exited    Update UTMP about System Boot/Shutdown
systemd-user-sessions.service                                                               loaded active exited    Permit User Sessions
systemd-vconsole-setup.service                                                              loaded active exited    Setup Virtual Console
tuned.service                                                                               loaded active running   Dynamic System Tuning Daemon
-.slice                                                                                     loaded active active    Root Slice
system-getty.slice                                                                          loaded active active    system-getty.slice
system-selinux\x2dpolicy\x2dmigrate\x2dlocal\x2dchanges.slice                               loaded active active    system-selinux\x2dpolicy\x2dmigrate\x2dlocal\x2dchanges.slice
system.slice                                                                                loaded active active    System Slice
user-0.slice                                                                                loaded active active    User Slice of root
user.slice                                                                                  loaded active active    User and Session Slice
dbus.socket                                                                                 loaded active running   D-Bus System Message Bus Socket
systemd-initctl.socket                                                                      loaded active listening /dev/initctl Compatibility Named Pipe
systemd-journald.socket                                                                     loaded active running   Journal Socket
systemd-shutdownd.socket                                                                    loaded active listening Delayed Shutdown Socket
systemd-udevd-control.socket                                                                loaded active running   udev Control Socket
systemd-udevd-kernel.socket                                                                 loaded active running   udev Kernel Socket
basic.target                                                                                loaded active active    Basic System
cryptsetup.target                                                                           loaded active active    Local Encrypted Volumes
getty.target                                                                                loaded active active    Login Prompts
local-fs-pre.target                                                                         loaded active active    Local File Systems (Pre)
local-fs.target                                                                             loaded active active    Local File Systems
multi-user.target                                                                           loaded active active    Multi-User System
network-online.target                                                                       loaded active active    Network is Online
network-pre.target                                                                          loaded active active    Network (Pre)
network.target                                                                              loaded active active    Network
paths.target                                                                                loaded active active    Paths
remote-fs.target                                                                            loaded active active    Remote File Systems
slices.target                                                                               loaded active active    Slices
sockets.target                                                                              loaded active active    Sockets
swap.target                                                                                 loaded active active    Swap
sysinit.target                                                                              loaded active active    System Initialization
timers.target                                                                               loaded active active    Timers
systemd-tmpfiles-clean.timer                                                                loaded active waiting   Daily Cleanup of Temporary Directories
```

## BCLinux 的日志

```bash
root@MiWiFi-RA70-srv ~]# systemctl status 
● MiWiFi-RA70-srv
    State: degraded
     Jobs: 0 queued
   Failed: 1 units
    Since: Tue 2023-08-29 23:12:23 CST; 42s ago
   CGroup: /
           ├─user.slice
           │ └─user-0.slice
           │   ├─session-3.scope
           │   │ ├─1133 sshd: root [priv]
           │   │ ├─1135 sshd: root@pts/0
           │   │ ├─1136 -bash
           │   │ └─1183 systemctl status
           │   ├─session-1.scope
           │   │ ├─ 551 login -- root
           │   │ └─1081 -bash
           │   └─user@0.service
           │     └─init.scope
           │       ├─1072 /usr/lib/systemd/systemd --user
           │       └─1074 (sd-pam)
           ├─init.scope
           │ └─1 /usr/lib/systemd/systemd --switched-root --system --deserialize 16
           └─system.slice
             ├─rngd.service
             │ └─445 /sbin/rngd -f
             ├─systemd-networkd.service
             │ └─510 /usr/lib/systemd/systemd-networkd
             ├─systemd-udevd.service
             │ └─414 /usr/lib/systemd/systemd-udevd
             ├─polkit.service
             │ └─443 /usr/lib/polkit-1/polkitd --no-debug
             ├─chronyd.service
             │ └─435 /usr/sbin/chronyd
             ├─tuned.service
             │ └─543 /usr/bin/python3 -Es /usr/sbin/tuned -l -P
             ├─systemd-journald.service
             │ └─396 /usr/lib/systemd/systemd-journald
             ├─sshd.service
             │ └─538 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
             ├─crond.service
             │ └─549 /usr/sbin/crond -n
             ├─NetworkManager.service
             │ ├─508 /usr/sbin/NetworkManager --no-daemon
             │ ├─759 /sbin/dhclient -d -q -sf /usr/libexec/nm-dhcp-helper -pf /var/run/NetworkManager/dhclient-ens18.pid -lf /var/lib/NetworkManager/dhclient-d43b7a46-0dff-9d53-1068-ccc58c977db3-ens18.lease -cf /var/lib/NetworkManager/dhclient-ens18.conf ens18
             │ └─897 /sbin/dhclient -d -q -6 -N -sf /usr/libexec/nm-dhcp-helper -pf /var/run/NetworkManager/dhclient6-ens18.pid -lf /var/lib/NetworkManager/dhclient6-d43b7a46-0dff-9d53-1068-ccc58c977db3-ens18.lease -cf /var/lib/NetworkManager/dhclient6-ens18.conf ens18
             ├─systemd-hostnamed.service
             │ └─532 /usr/lib/systemd/systemd-hostnamed
             ├─rsyslog.service
             │ └─801 /usr/sbin/rsyslogd -n -i/var/run/rsyslogd.pid
             ├─firewalld.service
             │ └─485 /usr/bin/python3 /usr/sbin/firewalld --nofork --nopid
             ├─dbus.service
             │ └─431 /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation --syslog-only
             └─systemd-logind.service
               └─477 /usr/lib/systemd/systemd-logind
```

failed: Lm_sensors是一个命令行工具，用于显示所有芯片传感器数据的当前读数，包括CPU温度。

```
             ├─rngd.service
             ├─.service
             ├─.service
             ├─.service
             ├─.service
             ├─tuned.service

             ├─.service
             ├─.service
             ├─systemd-hostnamed.service
             ├─.service
             ├─.service
             ├─.service
             └─.service
```



|服务名称|Centos7.8|BClinux oe21.10|
|-|:-:|:-:|
|rsyslog||rsyslog|
|sshd|Y|sshd|
|postfix|||
|tuned|Y|tuned|
|NetworkManager||systemd-networkd，NetworkManager|
|firewalld||firewalld|
|crond||crond|
|systemd-logind||systemd-logind|
|polkit||polkit|
|dbus||dbus|
|qemu-guest-agent|||
|auditd|||
|systemd-udevd||systemd-udevd|
|systemd-journald||systemd-journald|
|-||rngd|
|||systemd-hostnamed|
|||chronyd|




---

## 参考文献

- [Systemd 官方文档](https://www.freedesktop.org/software/systemd/man/systemd.service.html)
- [Systemd 服务管理教程](https://cloud.tencent.com/developer/article/1516125)
- [一次搞定 Linux systemd 服务脚本](https://zhuanlan.zhihu.com/p/651550778)
- [Cgroups 与 Systemd](https://www.cnblogs.com/sparkdev/p/9523194.html)
- [金步国作品集 - Linux](https://www.jinbuguo.com/)
- [Linux Kdump](https://zhuanlan.zhihu.com/p/74319084)