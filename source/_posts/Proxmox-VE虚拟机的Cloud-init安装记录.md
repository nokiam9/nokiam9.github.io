---
title: Proxmox VE虚拟机的Cloud-init安装记录
date: 2020-08-02 10:31:45
tags:
---

## 安装步骤

1. 安装Centos 7.8操作系统。

   {% asset_img disk-partition.png %}

   > 系统装完后，必须将网卡配置文件内的onboot打开，清除uuid！！！

2. 关闭selinux和firewalld以及碍事的NetworkManager

    ``` sh
    # 关闭Selinux
    sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
    setenforce 0

    # 关闭Firewalld
    systemctl disable --now firewalld

    # 关闭NetworManager
    systemctl disable --now NetworkManager

    # 设置Linunx内核，允许IP转发
    modprobe br_netfilter
    echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
    echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.conf
    echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.conf
    sysctl -p
    ```

3. 安装必要的虚拟化软件，为了让虚拟化层可以重启和关闭虚拟机，必须安装acpid服务；为了使根分区正确调整大小安装cloud-utils-growpart，cloud-init支持下发前设置信息写入。

    ``` sh
    yum install -y acpid cloud-init cloud-utils-growpart
    yum install -y git wget yum-utils net-tools bind-utils
    systemctl enable acpid

    echo "NOZEROCONF=yes" >> /etc/sysconfig/network

    sed -ri '/UseDNS/{s@#@@;s@\s+.+@ no@}' /etc/ssh/sshd_config
    systemctl restart sshd
    ```

4. 设置cloud-init
    设置允许root登录，允许输入口令，禁止第一次启动后yum更新软件

    ``` sh
    sed -ri '/disable_root/{s#\S$#0#}' /etc/cloud/cloud.cfg
    sed -ri '/ssh_pwauth/{s#\S$#1#}' /etc/cloud/cloud.cfg
    sed -ri '/package-update/s@^@#@' /etc/cloud/cloud.cfg
    ```

    默认cloud-init会创建一个系统类型的用户，手工编辑配置文件取消掉。

    ``` conf
    #  default_user:
    #    name: centos
    #    lock_passwd: true
    #    gecos: Cloud User
    #    groups: [wheel, adm, systemd-journal]
    #    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
    #    shell: /bin/bash
    ```

5. 虚拟机关机，并在PVE控制台上将其转换为模版templete

## 使用Cloud-init模版克隆新的虚拟机

## 参考文献

- [proxmox中cloud-init使用方法](https://kinkinlu.com/2019/04/18/proxmox%E4%B8%ADcloud-init%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/)
- [Cloud-init的基本原理](https://xixiliguo.github.io/post/cloud-init-1/)
- [CentOS 7 下 yum 安装和配置 NFS](https://qizhanming.com/blog/2018/08/08/how-to-install-nfs-on-centos-7)

---

