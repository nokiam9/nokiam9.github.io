---
title: Centos宿主机如何安装docker最新版本
date: 2019-10-26 14:55:27
tags:
---

## 问题现象

测试成功的APP软件包，但在宿主机上安装失败，显示docker container的DNS不成功，判断是宿主机的Docker版本太低，对Docker DNS的支持不好。  

宿主机是Centos, 内核版本`3.10.0-1062.4.1.el7.x86_64`, 使用`yum install docker`安装docker，docker的最新版本号`1.13`，即使采用`yum update docker`也无法更新。  

## 原因分析

Docker版本多次演进，现在分成社区版和企业版，安装方式已经改变。

## 解决方案

1. 删除旧版本

    ``` bash
    $ sudo yum remove docker \
                    docker-client \
                    docker-client-latest \
                    docker-common \
                    docker-latest \
                    docker-latest-logrotate \
                    docker-logrotate \
                    docker-engine
    ```

2. 设置安装源

    ``` bash
    $ sudo yum install -y yum-utils \
        device-mapper-persistent-data \
        lvm2

    $ sudo yum-config-manager \
        --add-repo \
        https://download.docker.com/linux/centos/docker-ce.repo
    ```

3. 安装最新版docker

    ``` bash
    $ sudo yum install docker-ce docker-ce-cli containerd.io
    ...

    ```

    > If prompted to accept the GPG key, verify that the fingerprint matches 060A 61C5 1B55 8A7F 742B 77AA C52F EB6B 621E 9F35, and if so, accept it.

    现在`$ docker -v`检查版本，显示`Docker version 19.03.4, build 9013bf583a`，版本升级成功！！！

4. 检查docker历史版本的信息
  
    ``` bash
    $ yum list docker-ce --showduplicates | sort -r
    docker-ce.x86_64            3:19.03.4-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:19.03.4-3.el7                    @docker-ce-stable
    docker-ce.x86_64            3:19.03.3-3.el7                    docker-ce-stable  
    docker-ce.x86_64            3:19.03.2-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:19.03.1-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:19.03.0-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:18.09.9-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:18.09.8-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:18.09.7-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:18.09.6-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:18.09.5-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:18.09.4-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:18.09.3-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:18.09.2-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:18.09.1-3.el7                    docker-ce-stable
    docker-ce.x86_64            3:18.09.0-3.el7                    docker-ce-stable
    docker-ce.x86_64            18.06.3.ce-3.el7                   docker-ce-stable
    docker-ce.x86_64            18.06.2.ce-3.el7                   docker-ce-stable
    docker-ce.x86_64            18.06.1.ce-3.el7                   docker-ce-stable
    docker-ce.x86_64            18.06.0.ce-3.el7                   docker-ce-stable
    docker-ce.x86_64            18.03.1.ce-1.el7.centos            docker-ce-stable
    docker-ce.x86_64            18.03.0.ce-1.el7.centos            docker-ce-stable
    docker-ce.x86_64            17.12.1.ce-1.el7.centos            docker-ce-stable
    docker-ce.x86_64            17.12.0.ce-1.el7.centos            docker-ce-stable
    docker-ce.x86_64            17.09.1.ce-1.el7.centos            docker-ce-stable
    docker-ce.x86_64            17.09.0.ce-1.el7.centos            docker-ce-stable
    docker-ce.x86_64            17.06.2.ce-1.el7.centos            docker-ce-stable
    docker-ce.x86_64            17.06.1.ce-1.el7.centos            docker-ce-stable
    docker-ce.x86_64            17.06.0.ce-1.el7.centos            docker-ce-stable
    docker-ce.x86_64            17.03.3.ce-1.el7                   docker-ce-stable
    docker-ce.x86_64            17.03.2.ce-1.el7.centos            docker-ce-stable
    docker-ce.x86_64            17.03.1.ce-1.el7.centos            docker-ce-stable
    docker-ce.x86_64            17.03.0.ce-1.el7.centos            docker-ce-stable
    ```

## 参考文档

[Centos安装docker的官方文档](https://docs.docker.com/install/linux/docker-ce/centos/)
