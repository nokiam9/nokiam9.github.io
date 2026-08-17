---
title: Proxmox Backup Server 部署概述
date: 2026-08-17 23:40:54
tags:
---

Proxmox 备份服务器是一款面向企业级环境的开源备份解决方案，专为虚拟机、容器及物理主机的高效备份与恢复而设计，具有以下特点：

- 与 Proxmox VE 实现深度集成，为虚拟机和容器提供高效、无缝的备份解决方案，适用于本地环境及跨远程站点的统一数据保护
- 采用增量备份机制，仅将客户端发生变化的数据块传输至服务器端，并在接收时执行去重处理
- 采用 ZSTD 压缩算法，具备卓越的数据压缩能力，可在单秒内完成数吉字节数据的压缩处理
- 采用客户端-服务器架构，实现备份系统的逻辑分离，支持多个独立主机共享同一备份服务器资源，支持将本地数据存储同步至远程位置
- 提供原生集成的图形化管理界面（GUI），所有管理操作均可通过标准Web浏览器访问（https://:8007）完成

## 一、软件安装

官方优先推荐使用 PBS ISO 全新安装，[ISO 下载页面](https://enterprise.proxmox.com/iso/)显示，当前最新版本是 4.2。
在已有 Debian 上用 apt 安装属于高级方式，需要自行处理 ZFS、存储、网络配置等，但考虑到一台裸机只做备份服务器太奢侈了，我们就高级了一把！采用在 Proxmox VE 8.4 宿主机的 apt 在线安装，实际版本为 3.4.9，因为这是 Debian12 的最新版本。

> PBS 4.x 基于 Debian 13 (Trixie)；PBS 3.x 基于 Debian12 (Bookworm)

```bash
#导入gpg密钥
wget https://enterprise.proxmox.com/debian/proxmox-release-bookworm.gpg -O /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg

#添加源
echo "deb http://download.proxmox.com/debian/pbs bookworm pbs-no-subscription" > /etc/apt/sources.list.d/pbs.list

apt update
apt install proxmox-backup-server -y
```

当前最新版本是 4.2-1，本次安装的版本是 3.4.9。

## 二、部署方案

实际备份部署方案如下：

1. 2台 Proxmox VE 宿主机同时安装 Proxmox Backup Server，均可提供数据备份能力；
2. PBS-130 作为主备份服务器，内置 1TB STAT 硬盘作为存储卷；
3. Proxmox Datecenter 新增 Proxmox Backup Server = PBS-130，并新增 Backup 任务，设置为每日3点备份所有 VM 和 CT；
4. PBS-132 作为从备份服务器，USB外接 1TB STAT 硬盘作为存储卷；
5. PBS-130 新增 Access Control ，定义用户 syncuser@pbs 并设置 Datastore Reader 权限；
6. PBS-132 存储卷新增 Sync Jobs，设置每日5点从 PBS-130 **Pull** 获取 Datastore 数据存入本地硬盘。

注意事项：

- 为数据备份定时设置 Verify ，确保数据安全性
- 2台 PVE 宿主机均兼作 PBS，拥有一个内置硬盘备份 + 一个USB硬盘备份，满足数据备份的 3-2-1 原则！

---

## 参考文献

- [Proxmox Backup Server 官方文档](https://pbs.proxmox.com/docs/)
- [Proxmox Backup Server 技术路标](https://pbs.proxmox.com/wiki/Roadmap)
- [Proxmox Backup Server 完全指南：增量备份、去重与自动化策略](https://shallowhave.github.io/p/proxmox-backup-server-guide/)
- [用 Proxmox Backup Server 备份 PVE 虚拟机|容器教程](https://blog.zwbcc.cn/archives/022069b8-6436-488c-9c64-3e9fde02b4c6)
