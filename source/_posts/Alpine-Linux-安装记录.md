---
title: Alpine Linux 安装记录
date: 2025-02-16 11:24:31
tags:
---

## 一、概述

下载页面：[https://www.alpinelinux.org/downloads/](https://www.alpinelinux.org/downloads/)

- Stanard：基础版本，安装需要网络支持
- Extended：最常用的版本，适合路由器和服务器。包含了常用软件包，仅支持 x86 架构
- Netboot：网络启动版本，适合 diskless 启动模式
- Rasperry PI：树莓派专用版本
- Mini Root Filesystem：最小化启动版本，适用于容器镜像
- Virtual：与基础版本类似，为虚拟机优化，系统内核较小
- Generic U-Boot：提供 LTS 内核，包含 U-Boot 启动器
- XEN：支持 XEN 虚拟化

还有为云计算的定制版本：[https://www.alpinelinux.org/cloud/](https://www.alpinelinux.org/cloud/)

亚马逊定制的 AWS 是 `Release` 版本，宣称进行了充分测试。
微软 Azure、谷歌 GCP、甲骨文 OCI 的版本是 `Beta` 版，此外还有 NoCloud 的版本；
最新发布的 Alpine 版本还提供一个 Generic 版本，也就是 `Alpha` 版本。

具体又分为几个规格，可以根据需要选择下载文件.

- CPU 架构：x86/64 & aarch64
- 启动方式：BIOS & UEFI
- 预装软件：Cloud-init & Tiny Cloud
- 设备类型：虚拟机 Virtual & 裸金属 Metal

本文基准版本：[nocloud_alpine-3.21.2-x86_64-bios-cloudinit-r0.qcow2](https://dl-cdn.alpinelinux.org/alpine/v3.21/releases/cloud/nocloud_alpine-3.21.2-x86_64-bios-cloudinit-r0.qcow2)

- 没有 bash，而是采用 busybox 集成了几十个常用命令，参数和输出都很精简
- 没有安装 sudo，临时使用可以用 doas 命令替代
- 系统服务不是 systemd，而是传统的 rc.service
- 软件源管理不是 apt 或 yum，而是自定义的 apk
- 语言编译器不是 gcc，而是更简单的 glibc

## 二、定制步骤

基准版本已经预装 cloudinit，由于采用 qcow2 格式而非 ISO 格式，新建 VM 需在 PVE Console 通过 `qm importdisk` 加载。
正常启动后基础镜像仅有 200M 左右，默认用户是 alpine，密码在 cloudinit CDROM 设置。

基线配置要点：

- 默认 HD 约 200M，一般扩容 500M 即可
- 激活 root 并设置 password
- 安装常用工具：wget git net-tools bind-tools
- 安装虚拟机软件：qemu-guest-agent，并激活服务和设置自启动
- 设置时区：sudo apk add tzdata；sudo ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
- SSH 配置: PermitRootLogin : false？

### 1. 设置软件源

编辑 `/etc/apk/repositories`

```yaml
#/media/cdrom/apks
http://mirrors.aliyun.com/alpine/v3.20/main
#http://mirrors.aliyun.com/alpine/v3.20/community
```

### 2. 常用指令

```bash
poweroff                                # 关机指令，替代 shutdown now

apk update                              # 更新软件源索引
apk add qemu-guest-agent                # 安装软件包

rc-service qemu-guest-agent start       # 系统服务启动软件包，start｜stop｜restart｜status
rc-update add qemu-guest-agent boot     # 设置系统自启动服务
rc-status                               # 列出运行级的状态管理。
```

---

## 附录一：ISO 安装方式

CDROM 安装启动是在 RAM 进行的，所以速度特别快，但**无法存储系统配置**，登录后提示使用 `setup-alpine` 进行安装。

- Keymap：us us
- Hostname：默认 localhost
- Interface：默认 DHCP
- Root Password：******
- Timezone：PRC
- Proxy：默认 none
- APK Mirror：清华源 14，阿里云 49
- User：默认，无普通用户；默认不允许 Root 登录；使用 sshd 软件包
- Disk & Install：选择 sda；模式选择 lvmsys，即 LVM 管理的 system 盘

格式化硬盘后，提示安装完成，可以 reboot 了。

---

## 参考文献

- [PVE 安装 Alpine + Cloud-init](https://www.techtutorials.tv/sections/promox/proxmox-alpine-cloud-init-image/)
- [Alpine Linux 安装](https://www.cnblogs.com/smlile-you-me/p/17321107.html)
- [ALpine 踩坑指南](https://www.cnblogs.com/sunsky303/p/11548343.html)

## 官方文档

- [Alpine 安装手册](https://wiki.alpinelinux.org/wiki/Installation#)
