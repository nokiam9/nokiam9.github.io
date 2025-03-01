---
title: Debian12 Cloud 安装记录
date: 2025-03-01 23:57:04
tags:
---

Debian 的每个版本都有绰号，都来自于电影《玩具总动员》中的角色名。

- Debian 8：Jessie，一个乐于助人，热心而温柔的女牛仔
- Debian 9：Stretch，紫色的大章鱼
- Debian 10：Buster：臭小子，安迪一家养的小狗
- Debian 11：Bullseye，红心，风驰电掣的玩具马
- Debian 12：bookworm，书虫，手持强力手电筒，帮助你阅读

随着云计算的普及，在传统 ISO 镜像安装方式的基础上，Debian 还提供了 Cloud 版本的官方支持，请参见[https://cloud.debian.org/images/cloud/](https://cloud.debian.org/images/cloud/) ，子版本有以下类型：

- azure: 基于 Microsoft Azure 环境优化
- ec2: 基于 Amazon EC2 环境优化
- generic: 基于 OpenStack 环境，预装 cloud-init，也可以用于裸金属服务器
- genericcloud: 与 generic 相似，系统更精简，但不支持额外硬件设备
- nocloud: 单机版本，不提供 cloud-init，默认无密码的 root 用户

本次安装使用最新的 Debian 12 generic-cloud 版本，64位x86 架构，2025年2月10日发布。
镜像下载地址：[debian-12-genericcloud-amd64-20250210-2019.qcow2](https://cloud.debian.org/images/cloud/bookworm/20250210-2019/debian-12-genericcloud-amd64-20250210-2019.qcow2)
预装软件信息：[debian-12-genericcloud-amd64-20250210-2019.json](https://cloud.debian.org/images/cloud/bookworm/20250210-2019/debian-12-genericcloud-amd64-20250210-2019.json)

在 PVE console 导入 qcow2 镜像，初始磁盘空间为 3GB。
> 为了替换无比丑陋的 NoVNC，VM 硬件添加 serial port 以支持 xterm.js

配置 cloud-init，**默认用户 debian 和初始密码**，取消软件自动升级参数等。。。

## 一、设置国内镜像源

系统源配置文件不再是传统的 `/etc/apt/sources.list`，而是改为 Deb 822 格式。
配置文件：`/etc/apt/sources.list.d/debian.sources`

```yml
Types: deb deb-src
URIs: https://mirrors.aliyun.com/debian             # mirror+file:///etc/apt/mirrors/debian.list 
Suites: bookworm bookworm-updates bookworm-backports
Components: main
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb deb-src
URIs: https://mirrors.aliyun.com/debian-security    # mirror+file:///etc/apt/mirrors/debian-security.list
Suites: bookworm-security
Components: main
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
```

如果需要使用非自由固件，则在编辑配置时需要添加 `non-free-firmware`。

## 二、安装常用软件

```bash
apt update

# 设置时区
timedatectl set-timezone Asia/Shanghai
date

# 安装常用软件
apt install -y git wget net-tools bind9-dnsutils

# 安装 qemu 相关软件
apt install -y qemu-guest-agent acpid 
systemctl start qemu-guest-agent
```

### 关闭 ipv6

编辑 `/etc/sysctl.conf`，追加以下配置信息：

```txt
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
net.ipv6.conf.eth0.disable_ipv6 = 1
```

执行下面命令，以使配置生效：
`sysctl -p`

现在基础安装完成！大约需要 400M 磁盘空间。

```consoel
Filesystem      Size  Used Avail Use% Mounted on
udev            462M     0  462M   0% /dev
tmpfs            97M  568K   96M   1% /run
/dev/sda1       2.8G  1.2G  1.5G  46% /
tmpfs           481M     0  481M   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
/dev/sda15      124M   12M  113M  10% /boot/efi
tmpfs            97M     0   97M   0% /run/user/1000
tmpfs            97M     0   97M   0% /run/user/0
```

## 三、安装 Docker

### 1. 获得 docker 源公钥

```bash
apt-get update
apt-get install ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

cat /etc/apt/keyrings/docker.asc
```

如果 curl 成功的话，此时应该显示 docker 源公钥的信息.

### 2. 安装 docker 源

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update
```

此时应该可以看到 docker 源信息。
> docker 源的配置文件仍然是传统的单行模式，不是新的 Deb 822 格式。

### 3. 安装docker

```bash
# Install the Docker packages
apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
apt-get install docker-compose

docker info
```

此时应该看到 docker 启动的配置信息，当前版本是 28.0

### 4a. 设置本地镜像源

编辑 `/etc/docker/daemon.json`

```yaml
{
    "insecure-registries": ["harbor.lan"],
    "registry-mirrors": ["https://harbor.lan"]
}
```

重启 Docker 让配置生效

```bash
systemctl daemon-reload
systemctl restart docker

docker run harbor.lan/proxy/library/hello-world
```

如果拉取镜像成功的话，此时应该看到 hello-world 的运行信息

### 4b. 为 docker 提供 ssl proxy

新建 docker systemd 启动服务目录

```bash
mkdir -p /etc/systemd/system/docker.service.d
```

编辑 proxy 配置文件`/etc/systemd/system/docker.service.d/http-proxy.conf`

```yaml
[Service]
Environment="HTTPS_PROXY=http://192.168.73.5:1082"
```

重启 Docker 让配置生效

```bash
systemctl daemon-reload
systemctl restart docker

docker run hello-world
```

## 四、简要分析

### 1. 安装 generic 版本启动时，提示错误提示无法加载`regulatory.db`

原因分析：linux 内核加载 `cfg80211` 模块以支持无线网卡，但漏安装了系统软件。
解决办法： `apt install iw`

不过，现在是更加精简的 generic-cloud 版本，没有无线网卡模块也就没有这个错误了。

### 2. 系统启动时 linux 内核崩溃

这个版本还曾经出现系统崩溃，提示 kernel panic - not syncing: killing interrupt handler!
出现场景是 HD 扩容后重启时，但是再次重启就能正常处理！！！

### 3. 系统启动有 PCI 告警

提示信息：acpi PNP0A03:00: fail to add MMCONFIG information, can't access extended PCI configuration space under this bridge.
目前系统运行无影响，原因待查。

---

## 参考文献

- [PVE 制作基于 Cloud Init 的虚拟机模板](https://tech.he-sb.top/posts/creating-vm-template-for-pve-based-on-cloud-init/)
- [Ubuntu/Debian更换软件安装源](https://www.voidking.com/dev-ubuntu-change-software-repo/)
- [docker 设置代理，以及国内加速镜像设置](https://neucrack.com/p/286)
- [cfg80211：无法加载regulatory.db](https://forum.proxmox.com/threads/cfg80211-failed-to-load-regulatory-db.137298/)
- [为 Proxmox 虚拟机启用 SPICE 支持](https://cn.linux-terminal.com/?p=4467#google_vignette)
- [不同场景下 Docker 的网络代理方法](https://cloud.tencent.com/developer/article/2423782)

## 官方文档

- [SOURCES.LIST - Debian](https://manpages.debian.org/stretch/apt/sources.list.5.en.html)
- [Debian 系统如何安装 Docker - Docker 官方](https://docs.docker.com/engine/install/debian/)
