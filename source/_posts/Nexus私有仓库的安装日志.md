---
title: Nexus私有仓库的安装日志
date: 2021-02-14 16:24:15
tags:
---

## Nexus简介

Nexus是一个强大的Maven仓库管理器，它极大地简化了本地内部仓库的维护和外部仓库的访问。 如果使用了公共的Maven仓库服务器，可以从Maven中央仓库下载所需要的构件（Artifact），但这通常不是一个好的做法。
正常做法是在本地架设一个Maven仓库服务器，即利用Nexus可以只在一个地方就能够完全控制访问和部署在你所维护仓库中的每个Artifact。 Nexus在代理远程仓库的同时维护本地仓库，以降低中央仓库的负荷,节省外网带宽和时间，Nexus就可以满足这样的需要。
Nexus是一套“开箱即用”的系统不需要数据库，它使用文件系统加Lucene来组织数据。
Nexus使用ExtJS来开发界面，利用Restlet来提供完整的REST APIs，通过m2eclipse与Eclipse集成使用。
Nexus支持WebDAV与LDAP安全身份认证。
Nexus还提供了强大的仓库管理功能，构件搜索功能，它基于REST，友好的UI是一个extjs的REST客户端，它占用较少的内存，基于简单文件系统而非数据库。

---

## 准备工作

必须的硬件条件：

- 内存 > 4GB
- 可用磁盘空间 > 4GB (建议独立数据磁盘，32G以上）
- 已安装JDK8（Maven不是必须的）

> Sonatype官方文档宣称必须使用Oracle JDK，但OpenJDK似乎也没问题，但不支持JDK9以上版本

## 安装步骤

1. 下载nexus软件包，当前版本`3.29.2`，安装包解压后有两个目录：
   - 程序目录`./nexus-3.29.2-02/`， 改名后迁移到/opt
   - 工作目录`./sonatype-work/nexus`，注意包含子目录，根据需要迁移或直接挂载数据盘
  
    > 也可以采用docker部署，镜像位于`sonatype/nexus3:3.29.2`，run可以设置port、volume

2. 创建nexus用户，并设置文件权限

    ``` bash
    # 新建nexus用户及用户组，nexus3不允许root启动
    groupadd nexus
    useradd -d /home/nexus -g nexus nexus

    # 将代码目录和数据目录赋权给nexus
    chown -R nexus:nexus /opt/nexus
    chown -R nexus:nexus /opt/sonatype-work
    ```

3. 设置NEXUS环境变量，编辑文件`/etc/profile.d/nexus.sh`

    ``` config
    #!/bin/bash
    NEXUS_HOME=/opt/nexus/
    PATH=$NEXUS_HOME/bin:$PATH
    export PATH NEXUS_HOME
    ```

    > 直接`cat`时，由于$语义替换会出错

4. 为Nexus设置运行用户名，编辑`/opt/nexus/bin/nexus.rc`
   并设置`run_as_user="nexus"`

5. 设置系统自启动服务，创建``，并填写

    ``` bash
    cat > /usr/lib/systemd/system/nexus.service <<- EOF
    [Unit]
    Description=Nexus daemon
    After=network.target

    [Service]
    Type=forking
    LimitNOFILE=65536
    ExecStart=/opt/nexus/bin/nexus start
    ExecStop=/opt/nexus/bin/nexus stop

    User=nexus
    Restart=on-abort

    [Install]
    WantedBy=multi-user.target
    EOF
    ```

    然后就是常规操作

    ``` bash
    systemctl daemon-reload
    systemctl enable --now nexus
    systemctl status nexus
    ```

## NEXUS仓库设置

通过浏览器访问[http://192.168.0.147:8081](http://192.168.0.147:8081)

初次访问登录时，需要设置admin的密码，初始密码在`/opt/sonatype-work/nexus3/admin.xxxx`文件中，一般设为`admin123`。

然后，就可以根据需要设置各类私服仓库了。

1. 设置docker-proxy
2. 设置docker-group
3. 设置Realms权限

## Client使用方法

1. 设置insecure-registries
2. 设置registry-mirrors
3. 重启docker服务

``` sh
[root@test ~]# more /etc/docker/daemon.json
{
    "insecure-registries":[
        "192.168.0.147:9001",
        "192.168.0.149"
    ],
    "registry-mirrors": [
        "http://192.168.0.147:9001"
    ]
}
```

待续...

---

## 参考文献

- [安裝 Nexus Repository Manager 作為 Docker Hub Mirror & Docker Image Proxy](https://godleon.github.io/blog/Nexus_Repository/docker-configure-proxy-with-nexus/)
- [Nexus上配置Docker仓库](https://www.0x0f0f.com/docker/create-docker-repository-in-nexus/create-docker-repository-in-nexus/)
- [使用 Nexus OSS 为 Docker 镜像提供代理/缓存功能](https://jenkins-zh.cn/wechat/articles/2020/05/2020-05-13-using-nexus-oss-as-a-proxy-cache-for-docker-images/)
- [Nexus 安装和配置](https://wiki.jikexueyuan.com/project/linux-in-eye-of-java/Nexus-Install-And-Settings.html)
- [maven私服nexus3.x环境配置](https://www.xncoding.com/2017/09/02/tool/nexus.html)
- [CentOS 7 下安装和配置 Sonatype Nexus 3.3](https://qizhanming.com/blog/2017/05/16/how-to-install-sonatype-nexus-oss-33-on-centos-7)
- [maven私服 nexus2.x工作目录解读](https://www.cnblogs.com/blaketairan/p/7136735.html)
- [使用harbor和nexus作为docker registry](https://juejin.cn/post/6844903781654593550)
- [Java版本号解读](https://blog.csdn.net/wq6ylg08/article/details/91351339)
