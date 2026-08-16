---
title: PVE QDevice 仲裁节点部署
date: 2026-08-16 16:34:41
tags:
---

2 节点 PVE Cluster 存在**脑裂**问题，必须部署 QDevice 仲裁节点以提供额外 1 票！至于 3 及以上奇数节点，则不建议部署 QDevice，因为这是一个单点风险。

QDevice Github：[https://github.com/corosync/corosync-qdevice](https://github.com/corosync/corosync-qdevice)，最新版本 3.0.4

QDevice 的资源需求很小，最低配置：1 核 + 256M 内存即可，树莓派、NAS设备、独立 VM、LXC、小型 VPS 均可，但绝对不应在 PVE node 上部署！

## 一、PVE 仲裁节点安装

本次部署环境在绿联云 NAS DXP4800，选择了 VM 部署方式，硬件配置：1C、1G、HD=3G。
> 实际上也有 docker 部署方案，但手工操作过多，操作风险很高因此放弃！

Qdevice 是开源软件，支持各种 Linux 平台，但 PVE 宿主机的 Debain 肯定是最合适的！

- 基于 debian-12-nocloud-amd64.qcow2 安装，默认用户root，无口令
- 1C、1G、默认HD=3G
- 设置系统自启动

### 1. 安装并配置 ssh

```bash
# 默认时区是美国
timedatectl set-timezone Asia/Shanghai

# 默认 root 密码为空
passwd 

# 安装 ssh 软件
apt install ssh net-tools

# 编辑`/etc/ssh/sshd_config`, 设置 PermitRootLogin yes
vi /etc/ssh/sshd_config

# 重新启动sshd
apt restart sshd
```

### 2. 设置静态IP地址

默认安装启动后的 IP 地址是 DHCP 方式，需要修改为固定IP。
编辑`/etc/netplan/90-default.yaml`：

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens3:
      dhcp4: false
      dhcp6: false
      addresses:
        - 192.168.31.134/24
      routes: 
        - to: default
          via: 192.168.31.1
      nameservers:
        addresses: [223.5.5.5, 8.8.8.8]
```

启动netplan：

```bash
# 检查语法，SDN错误忽略
netplan try

# 执行网络变更
netplan apply

# 检查 ip 地址
ip a
    ```

### 3. 安装 corosync-qnetd 软件包

```bash
apt update
apt install corosync-qnetd -y

systemctl enable --now corosync-qnetd

# 确认 qnetd 服务启动，并正常侦听 5403 端口
systemctl status corosync-qnetd
netstat -tulpn | grep 5403
```

## 二、PVE Cluster集群配置

### 1. 安装 Qdevice 软件包

已有两台 PVE 宿主机，通过 UI 可以新建集群，当然`pvecm create caogo-cluster`也可以。
注意！两台宿主机都必须安装 QDevice 软件包：

```bash
# 安装软件包
apt update && apt install corosync-qdevice -y 
```

### 2. 在任意一个节点配置 Qdevice

```bash
# 主命令，有很多网络和安装操作步骤。。。
pvecm qdevice setup 192.168.31.134 -f 

# 验证集群与仲裁状态：
pvecm status

# 查看 QDevice 的实时底层连接状态
corosync-qdevice-tool -s
```

### 3. Cluster 状态检查

Cluster 正常工作状态如下：

```console
root@nuc8i3:~# pvecm status
Cluster information
-------------------
Name:             caogo-cluster
Config Version:   3
Transport:        knet
Secure auth:      on

Quorum information
------------------
Date:             Sun Aug 16 17:09:58 2026
Quorum provider:  corosync_votequorum
Nodes:            2
Node ID:          0x00000002
Ring ID:          1.16
Quorate:          Yes

Votequorum information
----------------------
Expected votes:   3
Highest expected: 3
Total votes:      3
Quorum:           2  
Flags:            Quorate Qdevice 

Membership information
----------------------
    Nodeid      Votes    Qdevice Name
0x00000001          1    A,V,NMW 192.168.31.130
0x00000002          1    A,V,NMW 192.168.31.132 (local)
0x00000000          1            Qdevice

- A (Alive)：该节点存活且处于连接状态。
- NV (No-Vote)：备用不投票。当两台物理机都在线时已有 2 票（已满足 Quorum），仲裁节点主动弃权；一旦单节点宕机，该标志立即解除并投出第 2 票。
- NMW (No-Master-Wait)：仲裁无需等待指定主节点确认即可即时决策。

- QDevice (corosync-qdevice)：运行在 PVE 节点本地的仲裁客户端进程，负责收集本节点状态并向外部仲裁机同步。
- QNetd (corosync-qnetd)：运行在外部第三方设备（如绿联 NAS、Debian VPS）上的仲裁服务端，本身不存储虚拟机，仅充当“第三方投票人”。
```

## 三、重装的环境清理操作（可选）

1. 仲裁节点：清理证书

    ```bash
    rm -rf /etc/corosync/qnetd/nssdb 
    ```

2. 每台宿主机L：均需清理老证书

    ```bash
    rm -rf /etc/corosync/qdevice/net/nssdb rm -f /etc/pve/qdevice-net-node.p12 
    ```
