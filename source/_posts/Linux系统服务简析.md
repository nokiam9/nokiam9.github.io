---
title: Linux系统服务简析
date: 2023-08-29 22:06:52
tags:
---

## 一、概述

Systemd 是 Linux 系统中一个重要的系统和服务管理器，最早是为了代替传统的 SysV 初始化系统（init）而开发的，相较于传统 init，systemd 具有许多优势。例如支持并行启动，可同时启动多个服务，提高系统启动速度；引入了单一进程（PID 1）和 cgroups 技术，可以更好地管理系统和服务进程。目前，许多主流 Linux 发行版都采用了 systemd 作为其默认的初始化系统，包括 Ubuntu、Debian、Fedora、CentOS、Arch Linux 等。

Systemd 是一系列工具的集合，其作用也远远不仅是启动操作系统，它还接管了后台服务、结束、状态查询，以及日志归档、设备管理、电源管理、定时任务等许多职责，并支持通过特定事件（如插入特定 USB 设备）和特定端口数据触发的 On-demand（按需）任务。

![系统架构](arch.png)

总的来说，使用 systemd 可以更加简单灵活地管理各种系统服务，它提供了统一的命令行工具和配置文件格式，使得对系统和服务的管理更加一致和简化。用户可以通过 systemctl 命令来控制 systemd 系统和管理服务。
注意，systemd有自己的日志系统，称为journald 。它替换了sysVinit中的syslogd。

```sh
# 列出所有定义 systemd 服务
systemctl list-unit-files

# 列出启动失败的 systemd 服务
systemctl list-units --state failed

# 检查某个系统服务是否失败
systemctl is-failed [unit]

# 检查某个服务的运行状态
systemctl status [unit]

# 检查 本次 systemd 服务启动的全量日志
journalctl -b

# 检查某个服务的启动日志
journalctl -u [unit]
```

## 二、Centos 7.8的基线版本分析

以刚刚完成操作系统安装的服务器为例，可以通过 `systemcl status` 查看 systemd 系统服务基本配置，具体分为三个部分：

- `init.slice`：systemd 的根进程，进程号是 1 ！即所有用户空间进程的祖先进程
- `user.slice`：当前登录用户的全部会话进程，包括bash、login、sshd ...
- `system.slice`：当前所有系统服务进程，包括 service 名称及其启动的进程号

```console
● MiWiFi-RA70-srv
    State: degraded
     Jobs: 0 queued
   Failed: 1 units
    Since: 六 2023-09-09 16:57:15 CST; 54s ago
   CGroup: /
           ├─1 /usr/lib/systemd/systemd --switched-root --system --deserialize 22
           ├─user.slice
           │ └─user-0.slice
           │   ├─session-2.scope
           │   │ ├─1099 sshd: root@pts/0    
           │   │ ├─1103 -bash
           │   │ └─1118 systemctl status
           │   └─session-1.scope
           │     ├─ 488 login -- root     
           │     └─1080 -bash
           └─system.slice
             ├─rsyslog.service
             │ └─817 /usr/sbin/rsyslogd -n
             ├─postfix.service
             │ ├─1056 /usr/libexec/postfix/master -w
             │ ├─1057 pickup -l -t unix -u
             │ └─1058 qmgr -l -t unix -u
             ├─tuned.service
             │ └─814 /usr/bin/python2 -Es /usr/sbin/tuned -l -P
             ├─sshd.service
             │ └─813 /usr/sbin/sshd -D
             ├─NetworkManager.service
             │ ├─498 /usr/sbin/NetworkManager --no-daemon
             │ ├─623 /sbin/dhclient -d -q -sf /usr/libexec/nm-dhcp-helper -pf /var/run/dhclient-eth0.pid -lf /var/lib/NetworkManager/dhclient-3a3e2847-23bf-477e-87fc-a0e6356ef5d7-eth0.lease -cf /var/lib/NetworkManager/dhclient-eth0.conf eth0
             │ └─842 /sbin/dhclient -d -q -6 -N -sf /usr/libexec/nm-dhcp-helper -pf /var/run/dhclient6-eth0.pid -lf /var/lib/NetworkManager/dhclient6-3a3e2847-23bf-477e-87fc-a0e6356ef5d7-eth0.lease -cf /var/lib/NetworkManager/dhclient6-eth0.conf eth0
             ├─firewalld.service
             │ └─496 /usr/bin/python2 -Es /usr/sbin/firewalld --nofork --nopid
             ├─crond.service
             │ └─480 /usr/sbin/crond -n
             ├─systemd-logind.service
             │ └─475 /usr/lib/systemd/systemd-logind
             ├─polkit.service
             │ └─473 /usr/lib/polkit-1/polkitd --no-debug
             ├─dbus.service
             │ └─470 /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation
             ├─qemu-guest-agent.service
             │ └─469 /usr/bin/qemu-ga --method=virtio-serial --path=/dev/virtio-ports/org.qemu.guest_agent.0 --blacklist=guest-file-open,guest-file-close,guest-file-read,guest-file-write,guest-file-seek,guest-file-flush,guest-exec,guest-exec-status -F/etc/qemu-ga/fsfreeze-hook
             ├─auditd.service
             │ └─446 /sbin/auditd
             ├─systemd-udevd.service
             │ └─381 /usr/lib/systemd/systemd-udevd
             └─systemd-journald.service
               └─354 /usr/lib/systemd/systemd-journald
```

与内核密切相关，一般作为必须的基础服务：

- `rsyslog`: Rocket-fast System Logging Service，用于操作系统收集各种日志信息
- `tuned`：Dynamic System Tuning Daemon，监视系统组件运行状态，动态优化Linux 内核服务
- `polkit`：Authorization Manager，非特权用户会话与特权系统环境之间的协商者
- `dbus`：D-Bus System Message Bus，用于进程与内核、进程之间的通信总线
- `auditd`：Security Auditing Service，负责将Linux审计记录写入磁盘
- `sshd`：OpenSSH server daemon，SSH后台服务
- `systemd-udevd`：udev Kernel Device Manager，Linux默认的物理设备管理工具
- `systemd-journald`：Journal Service，systemd 的标准日志工具
- `systemd-logind`：Login Service，登录服务
- `crond`：Command Scheduler，定时任务调度服务

与应用相关，一般作为可选的服务：

- `postfix`：Postfix Mail Transport Agent，邮件发送服务，注意是内部mail，经常被手工屏蔽
- `firewalld`：dynamic firewall daemon，系统防火墙服务，通常被手工屏蔽
- `NetworkManager`：Network Manager，网络管理服务
- `qemu-guest-agent`：QEMU Guest Agent，虚拟机和宿主机的命令通道

注意！systemd 状态显示为 degraded，而非 runnning，说明有系统进程发生异常。
通过 `systemctl --state=failed` 检查发现是 kdump.service ，原因是操作系统安装时没选配置。

```console
  UNIT          LOAD   ACTIVE SUB    DESCRIPTION
● kdump.service loaded failed failed Crash recovery kernel arming
```

## 三、BCLinux 的基线版本分析

众所周知，BCLinux oe21.10 是基于 openEuler 21.10 的套娃版本，但也做了一些调整，检查结果如下：

删除的服务有：

- `auditd`
- `qemu-guest-agent`

增加的服务有：

- `rngd`：Hardware RNG Entropy Gatherer Daemon。使用环境噪声和硬件随机数生成器来生成熵，并存储到内核的随机数熵池
- `chronyd`：另一个版本的 NTP 时间服务器
- `systemd-networkd`：systemd 提供的网络管理工具。已有 NetworkManager ，这个可删除

除了 kdump 之前，还发现了 1 个异常的服务：

- `lm_sensors`: 检测CentOS系统的CPU温度，对于虚拟机没意义！

## 四、openEuler 22.03 的基线版本分析

与 BCLinux oe21.10 对比分析，可以发现：

- 直接删除了 `firewalld`
- 同样删除了 `postfix` ，但保留了 `auditd`
- 同样增加了 `rngd` 和 `chronyd` 的系统服务
- 保留了虚拟机的组件 `qemu-guest-agent`，但又增加了 `acpid`？
- 增加了用于 NFS 服务的组件 `gssproxy` 和 `rpcbind`，似乎并不合理？
- 增加了 `restorecond`，用于给 SELinux 监测和重新加载正确的文件上下文
- 增加了`systemd-hostnamed`，用于修改主机名称，似乎多余了！
- 网络管理软件仍然是 NetworkManager

## 五、腾讯云 Centos 7 的基线版本

与标准的 Centos 安装版本相比，有以下变化：

- 删除了 postdfix、firewalld，保留了 auditd
- 网络管理直接基于 cloud-init 的静态文件配置，不采用 NetworkManager !!!
- 启用基于 ntpd 的时间服务器
- 启用虚拟机电源管理的 acpid ，但没有 qemu-guest-agent

还有几个有意思的问题：

- 启用了一个类似 crond 的调度任务系统 atd ，很奇怪？
- 启用了 rhsmcertd：Red Hat Subscription Manager CERTification Daemon，红帽的订阅服务
- 启用了 libstoragemgmt，用于 ceph 等后端存储阵列管理
- 启用了 lvm2-lvmetad，用于 lvm2 的元数据管理，可能是安装 docker 引入的？
- 启用了 tat_agent：TencentCloud Automation Tools，腾讯开发的自动化助手

---

## 附录一：Systemd 的进程管理

系统中运行的所有进程，都是 systemd init 进程的子进程。在资源管控方面，systemd 提供了三种 unit 类型：

- service： 一个或一组进程，由 systemd 依据 unit 配置文件启动。service 对指定进程进行封装，这样进程可以作为一个整体被启动或终止。
- scope：一组外部创建的进程。由进程通过 fork() 函数启动和终止、之后被 systemd 在运行时注册的进程，scope 会将其封装。例如：用户会话、 容器和虚拟机被认为是 scope。
- slice： 一组按层级排列的 unit。slice 并不包含进程，但会组建一个层级，并将 scope 和 service 都放置其中。真正的进程包含在 scope 或 service 中。在这一被划分层级的树中，每一个 slice 单位的名字对应通向层级中一个位置的路径。

默认情况下，systemd 会自动创建 slice、scope 和 service unit 的层级(slice、scope 和 service 都是 systemd 的 unit 类型，来为 cgroup 树提供统一的层级结构。

- .slice：根 slice
- system.slice：所有系统 service 的默认位置
- user.slice：所有用户会话的默认位置
- machine.slice：所有虚拟机和 Linux 容器的默认位置

## 附录二：Systemd 的管理目录

Unit 文件按照 Systemd 约定，应该被放置指定的三个系统目录之一中。这三个目录是有优先级的，如下所示，越靠上的优先级越高。因此，在三个目录中有同名文件的时候，只有优先级最高的目录里的那个文件会被使用。

- `/etc/systemd/system`：系统或用户自定义的配置文件
- `/run/systemd/system`：软件运行时生成的配置文件
- `/usr/lib/systemd/system`：系统或第三方软件安装时添加的配置文件。

## 附录三：Systemd 的配置管理

CentOS7的服务systemctl脚本存放在:/usr/lib/systemd/,有系统（system）和用户（user）之分,需要开机不登陆就能运行的程序，存在系统服务里，即：/usr/lib/systemd/system目录下.
CentOS7的每一个服务以.service结尾，一般会分为3部分：[Unit]、[Service]和[Install]。

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

## 附录四：systemctl 的全量信息

```console
[root@MiWiFi-RA70-srv ~]# systemctl
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
  session-2.scope                                                                             loaded active running   Session 2 of user root
  auditd.service                                                                              loaded active running   Security Auditing Service
  crond.service                                                                               loaded active running   Command Scheduler
  dbus.service                                                                                loaded active running   D-Bus System Message Bus
  firewalld.service                                                                           loaded active running   firewalld - dynamic firewall daemon
  getty@tty1.service                                                                          loaded active running   Getty on tty1
● kdump.service                                                                               loaded failed failed    Crash recovery kernel arming
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

LOAD   = Reflects whether the unit definition was properly loaded.
ACTIVE = The high-level unit activation state, i.e. generalization of SUB.
SUB    = The low-level unit activation state, values depend on unit type.

84 loaded units listed. Pass --all to see loaded but inactive units, too.
To show all installed unit files use 'systemctl list-unit-files'.
```

---

## 参考文献

- [Systemd 官方文档](https://www.freedesktop.org/software/systemd/man/systemd.service.html)
- [Systemd 入门教程：命令篇 - 阮一峰](http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)
- [Systemd 服务管理教程](https://cloud.tencent.com/developer/article/1516125)
- [一次搞定 Linux systemd 服务脚本](https://zhuanlan.zhihu.com/p/651550778)
- [Cgroups 与 Systemd](https://www.cnblogs.com/sparkdev/p/9523194.html)
- [金步国作品集 - Linux](https://www.jinbuguo.com/)
- [Linux Kdump](https://zhuanlan.zhihu.com/p/74319084)
- [linux的 0号进程 和 1 号进程](https://www.cnblogs.com/alantu2018/p/8526970.html)
