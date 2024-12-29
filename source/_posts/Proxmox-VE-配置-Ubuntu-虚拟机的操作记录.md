---
title: Proxmox VE 配置 Ubuntu 虚拟机的操作记录
date: 2024-12-29 18:19:33
tags:
---

随着 CentOS 正式退出市场，许多软件的在线更新出现错误，Canonical 公司的 Ubuntu 的替代价值越来越明显，许多 AI 大模型的开发工作都是基于 Ubuntu，为此考虑为PVE 服务器增加 Ubuntu 的虚拟机模版。
Ubuntu 是基于 Debian 的开源发行版，与红帽的 Centos 软件包存在许多差异，而由于其通常用于桌面系统，软件包比较庞杂，作为服务器需要仔细进行剪裁。

Ubuntu 的主版本号就是其**首次发布的年份**，还有一个带形容词的动物别名，其生命周期通常为5年。

- Ubuntu 24.04.1 LTS（Noble Numbat，高贵的袋鼠）：最新版本，从稳定性角度暂不考虑
- Ubuntu 18.04 LTS（Bionic Beaver，仿生海狸）：常规支持已于2023年5月31日结束
- Ubuntu 20.04 LTS（Focal Fossa，马岛长尾狸猫）：2025年就要结束常规维护周期，算了。。。
- Ubuntu 22.04 LTS（Jammy Jellyfish，幸运水母）：就是幸运的你了！

Ubuntu 官方提供了 ISO 格式的发行版，分为桌面版、服务器版和用于物联网的 IoT 版，此外还有一种为云计算提供的 Cloud 镜像版（就是基于 Cloud-init 的 IMG 格式的基线版本），适配了几乎所有主流云计算厂商，当然也可以用于私人定制。

根据[Ubuntu Cloud Images](https://cloud-images.ubuntu.com/releases/)的信息，Ubuntu 22.04 经历多次补丁升级，最新版本发布于2024年12月17日。
最终确下载文件是 x86 架构镜像：[ubuntu-22.04-server-cloudimg-amd64.img](https://cloud-images.ubuntu.com/releases/22.04/release-20241217/ubuntu-22.04-server-cloudimg-amd64.img)，组件信息参见[manifest文件](https://cloud-images.ubuntu.com/releases/22.04/release-20241217/ubuntu-22.04-server-cloudimg-amd64.manifest)。

## 1. 新建虚拟机

在 PVE 中新建一个 VM，基本配置包括：

- VM 命名为`ubuntu22`，这里要记住`VM ID`！
- CDROM 无需加载任何介质，因为不再使用 ISO 镜像文件，而是后续直接导入 IMG 镜像文件
- **勾选`Qemu Agent`参数**，便于后续安装`qemu-guest-agent`，可以通过 PVE 控制台执行开关机等操作
- 硬盘默认设置即可，反正后面要删除！
- 处理器建议 1C，内存建议 1G；网络默认`vmbr0`即可，注意**取消`Firewall`防火墙**

## 2. 导入镜像文件

登录 PVE 主机。可以直接在 PVE 控制台的 node 节点执行 Shell 操作，默认就是 root 用户。

- `wget`网络下载 IMG 文件，并存储在用户目录（可以建个子目录）
- 使用`qm importdisk <vmid> <source> <storage>`命令，挂载到 VM 作为新增磁盘设备。默认存储路径是`local-lvm`

``` console
root@nuc5i3:~/cloud-img# qm importdisk 701 ubuntu-22.04-server-cloudimg-amd64.img local-lvm
importing disk 'ubuntu-22.04-server-cloudimg-amd64.img' to VM 701 ...
  WARNING: You have not turned on protection against thin pools running out of space.
  WARNING: Set activation/thin_pool_autoextend_threshold below 100 to trigger automatic extension of thin pools before they get full.
  Logical volume "vm-701-disk-1" created.
  WARNING: Sum of all thin volume sizes (<696.70 GiB) exceeds the size of thin pool pve/data and the size of whole volume group (<476.44 GiB).
transferred: 0 bytes remaining: 2361393152 bytes total: 2361393152 bytes progression: 0.00 %
transferred: 23613931 bytes remaining: 2337779221 bytes total: 2361393152 bytes progression: 1.00 %
transferred: 47464002 bytes remaining: 2313929150 bytes total: 2361393152 bytes progression: 2.01 %
。。。
transferred: 2331167319 bytes remaining: 30225833 bytes total: 2361393152 bytes progression: 98.72 %
transferred: 2354781251 bytes remaining: 6611901 bytes total: 2361393152 bytes progression: 99.72 %
transferred: 2361393152 bytes remaining: 0 bytes total: 2361393152 bytes progression: 100.00 %
transferred: 2361393152 bytes remaining: 0 bytes total: 2361393152 bytes progression: 100.00 %
Successfully imported disk as 'unused0:local-lvm:vm-701-disk-1'
```

## 3. 配置虚拟机

对该 VM 的`hardware`标签页面进行配置:

- 发现新设备`Unused Disk 0`，Add 该设备后发现尺寸大约 2.25GB，这就是我们手工加载的镜像文件。
    适当增加磁盘空间以便后续系统软件安装，但空间过多会占用磁盘并延缓启动速度，建议增加 2GB 即可
- Remove 默认的`CDROM`；Detach & Remove 默认的磁盘`Hard disk(scsi0)`
- Add 一个 Cloudinit 设备（类型是CDROM），默认存储路径也是`local-lvm`

处理结果就是这个样子
![VM](pic01.png)

## 4. 配置cloudinit

对该 VM 的`Cloud-init`标签页面进行配置:

- User：设置默认用户`ubuntu`。注意，Ubuntu 默认不直接使用`root`，而是将默认用户赋予`sudo`权限
- Password：**建议设置**。Ubuntu 默认不允许密码远程登录，只有 PVE Console 可以密码登录。
- DNS domain & DNS server：默认 host 配置即可，后续可以自定义
- SSH public key：编辑录入（多个）指定设备的公钥。以后维护均通过 SSH 远程免密登录。
- IP Config(eth0)：当前采用 DHCP，后续可以自定义

处理结果就是这个样子
![VM](pic02.png)

> 如果不设置默认用户的密码，虽然可以从指定设备 SSH 免密登录，但初次启动采用 DHCP 方式 IP 地址不固定，而 Qemu Agent 尚未安装也无法在 PVE 查看 IP 地址，因此最好还是设置一下。

## 5. 启动测试

注意！！！由于修改了磁盘配置，必须在`Options`标签页面的`Boot Order`项检查 VM 的启动配置，确保内置硬盘是第一启动顺序，否则 VM 无法启动就白费了！

![VM](pic03.png)

如果检查无误，就可以启动 VM 了，通过 PVE Console 可以查看启动日志。
通过 PVE Console 进行密码登录，并使用`ip a`命令查看 IP 地址，现在可以在指定设备使用 SSH 免密登录了。

```console
buntu@ubuntu22:~$ systemctl status
● ubuntu22
    State: running
     Jobs: 0 queued
   Failed: 0 units
    Since: Sun 2024-12-29 13:48:24 UTC; 23min ago
   CGroup: /
           ├─user.slice 
           │ └─user-1000.slice 
           │   ├─user@1000.service 
           │   │ └─init.scope 
           │   │   ├─1617 /lib/systemd/systemd --user
           │   │   └─1618 (sd-pam)
           │   ├─session-3.scope 
           │   │ ├─7117 sshd: ubuntu [priv]
           │   │ ├─7174 sshd: ubuntu@pts/0
           │   │ ├─7175 -bash
           │   │ ├─7204 systemctl status
           │   │ └─7205 pager
           │   └─session-1.scope 
           │     ├─ 712 /bin/login -p --
           │     └─1626 -bash
           ├─init.scope 
           │ └─1 /sbin/init
           └─system.slice 
             ├─packagekit.service                       # 为软件安装提供统一的 DBus 接口
             │ └─1654 /usr/libexec/packagekitd
             ├─systemd-networkd.service 
             │ └─527 /lib/systemd/systemd-networkd
             ├─systemd-udevd.service 
             │ └─369 /lib/systemd/systemd-udevd
             ├─cron.service 
             │ └─650 /usr/sbin/cron -f -P
             ├─polkit.service 
             │ └─764 /usr/libexec/polkitd --no-debug
             ├─networkd-dispatcher.service 
             │ └─659 /usr/bin/python3 /usr/bin/networkd-dispatcher --run-startup-triggers
             ├─multipathd.service                       # 多路径存储设备管理
             │ └─365 /sbin/multipathd -d -s
             ├─systemd-journald.service 
             │ └─328 /lib/systemd/systemd-journald
             ├─unattended-upgrades.service              # Ubuntu 系统升级服务
             │ └─729 /usr/bin/python3 /usr/share/unattended-upgrades/unattended-upgrade-shutdown --wait-for-signal
             ├─ssh.service 
             │ └─807 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
             ├─snapd.service                            # Ubuntu 应用商店服务
             │ └─663 /usr/lib/snapd/snapd
             ├─rsyslog.service 
             │ └─2063 /usr/sbin/rsyslogd -n -iNONE
             ├─systemd-resolved.service 
             │ └─529 /lib/systemd/systemd-resolved
             ├─dbus.service 
             │ └─652 @dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation --syslog-only
             ├─systemd-timesyncd.service 
             │ └─474 /lib/systemd/systemd-timesyncd
             └─systemd-logind.service 
               └─664 /lib/systemd/systemd-logind
```

## 6. 生成模版

### 安装 Qemu-guest-agent

Qemu-guest-agent 用于 PVE 管理虚拟机，例如控制台开关机、显示 IP 地址等监控信息。

```bash
sudo apt-get update
sudo apt-get install -y qemu-guest-agent
sudo systemctl start qemu-guest-agent
sudo reboot
```

> GA 安装必须通过 PVE Console，因为 SSH 远程登录无法显示字符 UI 界面将导致安装失败！
> 安装过程中提示，内核需要从`5.15.0-127-generic`升级为`5.15.0-130-generic`，重启`packagekit.service`服务后，使用`uname -r`命令，查看内核版本升级成功

### 设置中国时区 + 常用软件包

``` bash
sudo timedatectl set-timezone Asia/Shanghai
sudo apt install net-tools
```

### 模版制作

现在关闭 VM，随后将 VM 转换为模版固定下来，以后就可以克隆小鸡了！

---

## 附录：APT（Advanced Packaging Tool）简介

Debian 系发行版，包括 Ubuntu、Linux Mint 和 elementary OS 等，采用了 Debian 包管理系统，使用`.deb`文件来管理软件包。

- `dpkg` 是 Debian 包管理系统的核心，直接负责操作`.deb`文件，包含了`dpkg-split`、`dpkg-trigger`和`dpkg-divert`等一组命令，这些命令会也被`apt`和`apt-get`等更高级的工具调用。
- `apt-get`和`apt-cache`是比较**古早**的命令行工具，用于与包管理系统交互。其以稳定可靠著称，经常用于自动化任务，比如 Shell 脚本当中。
- 2014 年之后，`apt`取代`apt-get`成为所有基于 Debian 的 Linux 发行版的默认软件包管理器工具。它整合了 apt-get 和 apt-cache 的功能，语法更简单，输出也更友好，比如带有进度条和颜色编码。

apt 的软件源配置文件位于：`/etc/apt/sources.list`，核心内容如下：

```console
deb http://archive.ubuntu.com/ubuntu jammy main restricted
deb http://archive.ubuntu.com/ubuntu jammy-updates main restricted
deb http://archive.ubuntu.com/ubuntu jammy universe
deb http://archive.ubuntu.com/ubuntu jammy-updates universe
deb http://archive.ubuntu.com/ubuntu jammy multiverse
deb http://archive.ubuntu.com/ubuntu jammy-updates multiverse
deb http://archive.ubuntu.com/ubuntu jammy-backports main restricted universe multiverse
deb http://security.ubuntu.com/ubuntu jammy-security main restricted
deb http://security.ubuntu.com/ubuntu jammy-security universe
deb http://security.ubuntu.com/ubuntu jammy-security multiverse
```

### 系统管理

- `sudo apt-get update` ：更新 apt 软件源的索引信息
- `sudo apt-get upgrade` ：更新所有已安装软件的最新版本
- `sudo apt-get dist-upgrade`：升级系统
- `sudo apt-get clean & sudo apt-get autoclean`：清理无用的包
- `sudo apt-get check` ：检查是否有损坏的包依赖

### 安装和卸载

- `sudo apt-get install <package>=<version>` ：安装指定版本的软件包
- `sudo apt-get remove <package>`：删除软件包

### 检索分析

- `sudo apt-get list`：显示所有可用的软件包
- `sudo apt-get list --installed`：显示当前已安装的软件包
- `sudo apt-get source <package>`：下载该软件包的源代码
- `sudo apt-cache search <package>`： 搜索包
- `sudo apt-cache show <package>` ：获取包的相关信息，如说明，大小，版本。
- `sudo apt-cache depends <package>` ：分析某个软件包的依赖关系

---

## 参考文献

- [Ubuntu 各版本代号简介](https://blog.csdn.net/zhengmx100/article/details/78352773)
- [Making a Ubuntu 24.04 VM Template for Proxmox and CloudInit](https://github.com/UntouchedWagons/Ubuntu-CloudInit-Docs)
- [在 PVE 8 中使用 Cloud-init 初始化 ubuntu cloud-image 并创建模板](https://never666.uk/2107/)
- [在 PVE 8 环境中制作 Ubuntu 的 Cloud Init虚拟机模板](https://itan90.cn/archives/174/)
- [Ubuntu Server最小化安装指南](https://www.oryoy.com/news/ubuntu-server-zui-xiao-hua-an-zhuang-zhi-nan-you-hua-xi-tong-zi-yuan-yu-ti-sheng-bian-cheng-huan-jin.html)
- [如何使用自动安装编写和执行 Ubuntu 无人值守安装](https://cn.linux-console.net/?p=33717#google_vignette)
- [无用的知识：PackageKit](https://5long.github.io/post/packagekit.html)
- [ubuntu22.04通过netplan配置网络](https://www.cnblogs.com/laina/articles/17674155.html)
- [APT、apt-get、apt-cache 和 apt](https://juejin.cn/post/7028510908175351816)

### 官方文档

- [Ubuntu 官方主页](https://cn.ubuntu.com)
- [Ubuntu Cloud Images 发布版下载](https://cloud-images.ubuntu.com/releases/)
- [Cloud-init documentation](https://cloudinit.readthedocs.io/en/stable/index.html)
- [Proxmax Cloud-Init Support](https://pve.proxmox.com/wiki/Cloud-Init_Support)
