---
title: Harbor的安装记录
date: 2020-07-12 15:33:33
tags:
---

## 系统规划

镜像仓库作为Kubernets集群的重要基础设施，必须考虑在内网的互联网隔离问题。
Docker虽然自带了Registry，但是不能提供细粒度的权限设置，此外，在大型集群中为了解决镜像拉取的性能瓶颈，经常需要解决多级镜像的同步问题，因此Harbor已经成为事实标准，为此本文研究了如何在内网离线安装Harbor。

基线版本的信息如下：

    - Centos=7.8
    - Docker-CE=19.03.12
    - Docker-compose=1.21.2
    - Harbor=v1.10.3 

Harbor目前采用http方式，Admin URL：[http://192.168.0.130:7350](http://192.168.0.130:7350)
Harbor用户数据的存放目录：`/data/harbor`。为避免重装系统造成数据丢失，采用一个独立硬盘mount到/data。

## Harbor Server的安装步骤

1. 安装docker-ce、docker-compose

    通过阿里云的镜像加速服务，yum安装docker-ce，详细操作方式参见[Kubernetes集群的安装记录](https://blog.caogo.cn/2020/06/25/Kubernetes集群的安装记录/)

    docker-compose的标准安装方法是从docker.com下载，速度太慢无法忍受。
    还好，阿里云提供了[Docker-toolbox的下载地址](http://mirrors.aliyun.com/docker-toolbox/linux/)。
    注意：要手工改文件名，设置执行权限，并搬到PATH路径下。

2. 下载harbor安装包

    这里是[Harbor v1.10.3 下载地址](https://github.com/goharbor/harbor/releases/tag/v1.10.3)，解压后放在目录`/root/harbor/`下。
    离线方式的安装包有600M+，其中包含了全部所需的镜像文件，后续安装中通过`docker load`方式直接读取压缩包，就不需要联网了。

    > v1.10.2有个bug无法正常安装，表现是log容器启动时，爆出sudo权限过期，可能是基础镜像的问题)

3. 配置并安装harbor

    在启动目录`/root/harbor/`下，编辑Harbor配置文件`harbor.yml`，关键信息如下：

        ``` yaml
        hostname：192.168.0.130
        http：
        port：7350
        # https:
        data_volume: /data/harbor/
        ```

    运行命令`install.sh`，自动拉取压缩包的镜像文件，并生成`docker-compose.yml`，这就是以后的部署配置。
    最后，Harbor主目录的结构是这样的：

        ``` sh
        [root@dnsmasq harbor]# tree /root/harbor -L 2
        /root/harbor                    ## Harbor Sever的主目录
        ├── common
        │   └── config
        ├── common.sh   
        ├── docker-compose.yml          ## 最后生成的Depolyment配置文件
        ├── harbor.v1.10.3.tar.gz       ## 离线安装的镜像文件包，docker save && docker load
        ├── harbor.yml                  ## 初始化安装的配置文件
        ├── install.sh                  ## 安装入口程序
        ├── LICENSE
        └── prepare
        ```

4. 启动harbor并检查

    通过`docker-compose up -d --build`启动Harbor，然后就可以通过浏览器访问[http://192.168.0.130:7350](http://192.168.0.130:7350)
    在浏览器界面，输入用户名和密码，就可以看见镜像仓库的具体信息了。

## Client的使用方法

需要注意的是，Client的docker是独立的配置，如果不需要上传镜像，根本不用登录Harbor Server，直接设置mirror镜像就可以了。

    ``` sh
    cat > /etc/docker/daemon.json << EOF
    {
        "exec-opts": ["native.cgroupdriver=systemd"],
        "log-driver": "json-file",
        "log-opts": {
        "max-size": "100m"
        },
        "storage-driver": "overlay2",
        "insecure-registries": [
        "192.168.0.130:7350"
        ],
        "registry-mirrors": [
            "http://192.168.0.130:7350"
        ]
    }
    EOF

    systemctl daemon-reload
    systemctl restart docker
    docker info
    ```

当然，注意由于私有仓库是http模式，需要显示设置insecure-registries的不安全访问方式。
修改配置文件后，还需要重启守护进程和docker.service，然后你就可以正常使用私有仓库Harbor了！！！

    ``` sh
    [root@localhost docker]# docker login http://192.168.0.130:7350 -u admin -p xxxxxx
    WARNING! Using --password via the CLI is insecure. Use --password-stdin.
    WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
    Configure a credential helper to remove this warning. See
    https://docs.docker.com/engine/reference/commandline/login/#credentials-store

    Login Succeeded

    [root@localhost docker]# docker tag python:3.7-slim 192.168.0.130:7350/library/python:3.7-slim

    [root@localhost docker]# docker push 192.168.0.130:7350/library/python:3.7-slim
    The push refers to repository [192.168.0.130:7350/library/python]
    0c6163f2d025: Pushed
    361df01300cf: Pushed
    8f9ba0be9040: Pushed
    0bd71a837902: Layer already exists
    13cb14c2acd3: Layer already exists
    3.7-slim: digest: sha256:e0f6a4df17d5707637fa3557ab266f44dddc46ebfc82b0f1dbe725103961da4e size: 1370
    ```

现在admin界面，就可以找到这个镜像了，以后就可以直接本地拉取了。

{% asset_img admin.png %}

## 注意事项

### 自制的image push小工具（还要优化......）

    ``` sh
    cat > harbor-push.sh << EOF
    #!/bin/bash 

    count=1
    while read repo tag others   # 从docker images的输出中获得镜像信息，注意剔除第一行 
    do 
        src_image=$repo":"$tag
        dst_image=192.168.0.130:7350/library/$src_image

        echo "$count: $src_image is pushing..." >& 2
        
        echo docker tag $src_image $dst_image
        echo docker push $dst_image
        echo docker rmi $dst_image

        count=$(($count + 1))
    done
    EOF

    chmod a+x harbor-push.sh
    docker images |tail -n +2 |./harbor-push.sh 
    ```

### Harbor Server的启动和停止方式

由于Harbor是采用docker-compose方式启动的，因此关机之前最好手工停止服务，输入：

`cd /root/harbor && docker-compose down`

开机后，启动Harbor服务的方式也类似，执行命令:

`cd /root/harbor && docker-compose up -d --build`

## 参考资料

- [Harbor的官方网站](https://github.com/goharbor/harbor)
- [Image-syncer：阿里云提供的另一个镜像同步工具](https://github.com/AliyunContainerService/image-syncer/blob/master/README-zh_CN.md)
- [很全面的Harbor系统集成方法](https://blog.csdn.net/hxpjava1/article/details/79308890)
- [Harbor与Nginx的集成中发现的问题](https://blog.csdn.net/weixin_33736048/article/details/92953567?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-7.nonecase&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-7.nonecase)
