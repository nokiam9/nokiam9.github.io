---
title: Docker Mirror 容器镜像服务解析
date: 2021-01-31 08:53:09
tags:
---

在实际生产环境下，机器都不能访问外部互联网，或者访问互联网络的带宽有限，同时有大量的容器镜像需要从外部下载，如果每次开发、测试、部署时都需要从互联网下载容器镜像，则将占用大量的带宽而且效率较低。同时，在某些场景如物联网场景中，需要使用移动网络接入互联网，这时带宽可能是系统部署的瓶颈。

更为严重的问题是，Docker Hub公有云容器镜像服务对客户端有限流设置，当镜像拉取操作达到一定流量时，会导致服务无法使用。Docker Hub最新免费服务条款（11月1日生效），匿名用户每6小时只允许拉取100次，登录用户200次，只有付费用户不受此限制。

Mirror代理缓存仓库的作用就是为了以上问题，通过运行一个缓存仓库允许你在本地储存镜像，减少过多的通过互联网从`Docker Hub`拉取镜像，这个特性对于一些在他们环境中拥有数量庞大的Docker引擎的用户来说很有用。

## image命名规则

容器镜像（iamge）的命名规则是：`myregistrydomain:port/foo/bar`，其中包含了几个关键信息：HUB域名或IP地址、端口号、仓库名、软件名、标记名。
例如，命令`docker pull harbor.caogo.local:80/library/mongo:3.6`的含义是：从`harbor.caogo.local`服务器，通过http端口，获取liblary仓库的mongo软件镜像，标签为3.6

`docker.io`是Docker的官方HUB，其中`/library`是官方仓库所在位置，而`/calico`等则是开发者在官方HUB的自定义仓库。因此，Docker规定官方镜像可以采用省略所有前缀的简写方式，即：
`docker pull docker.io/library/ubuntu` == `docker pull ubuntu`

常用的镜像仓库HUB包括：

- `registry-1.docker.io`                    Docker官方镜像
- `k8s.gcr.io`                              Google Kubernetes系统镜像
- `quay.io`                                 重要的第三方镜像
- `registry.cn-hangzhou.aliyuncs.com`       阿里云的系统镜像

## docker pull 镜像拉取流程

根据[Pulling An Image 官方文档](https://docs.docker.com/registry/spec/api/#pulling-an-image)的解释：
> An “image” is a combination of a JSON manifest and individual layer files.

一个镜像image包含了以下要素：

- 名称 name：The name of the image.
- 标记 tag：The tag for this version of the image.
- 分层文件 fsLayers：A list of layer descriptors (including digest)
- 签名 signature：A JWS used to verify the manifest content

> image已经建立了行业标准 - OCI镜像规范，确保Docker、Podman等不同容器产品之间共享image，可以参见附录继续研究

docker pull的过程很复杂，包括鉴权、校验，下载、合并镜像层，解压缩等等，但最核心的是两个步骤：

1. 发送请求 `GET /v2/<name>/manifests/<reference>` ，获取镜像的mainfest清单文件
    reference可以是标记tag，或摘要digest。
2. 发送请求 `GET /v2/<name>/blobs/<digest>` ，获取镜像层文件

镜像的mainfest文件格式的示例为

``` json
{
   "name": <name>,
   "tag": <tag>,
   "fsLayers": [
      {
         "blobSum": <digest>
      },
      ...
    ]
   ],
   "history": <v1 images>,
   "signature": <JWS>
}
```

## Docker Registry 镜像注册服务

Registry是Docker提供的镜像注册服务，其设计目标是一套存储和分发镜像的处理机制，实现方式是在Docker引擎中运行一个来自`docker.io/library/registry`的实例。

需要注意的是，Registry管理镜像是基于文件系统的,数据存储目录位于实例所在容器的`/var/lib/registry`

``` console
/ # tree /var/lib/registry
/var/lib/registry
└── docker
    └── registry
        └── v2
            ├── blobs
            │   └── sha256
            │       ├── 36
            │       │   └── 36c3a913e62f77a82582eb7ce30d255f805c3d1e11d58e1f805e14d33c2bc5a5
            │       │       └── data
            │       ├── 43
            │       │   └── 43773d1dba76c4d537b494a8454558a41729b92aa2ad0feb23521c3e58cd0440
            │       │       └── data
            │       └── 5a
            │           └── 5a3ea8efae5d0abb93d2a04be0a4870087042b8ecab8001f613cdc2a9440616a
            │               └── data
            └── repositories
                └── myregistry
                    ├── _layers
                    │   └── sha256
                    │       ├── 43773d1dba76c4d537b494a8454558a41729b92aa2ad0feb23521c3e58cd0440
                    │       │   └── link
                    │       └── 5a3ea8efae5d0abb93d2a04be0a4870087042b8ecab8001f613cdc2a9440616a
                    │           └── link
                    ├── _manifests
                    │   ├── revisions
                    │   │   └── sha256
                    │   │       └── 36c3a913e62f77a82582eb7ce30d255f805c3d1e11d58e1f805e14d33c2bc5a5
                    │   │           └── link
                    │   └── tags
                    │       └── latest
                    │           ├── current
                    │           │   └── link
                    │           └── index
                    │               └── sha256
                    │                   └── 36c3a913e62f77a82582eb7ce30d255f805c3d1e11d58e1f805e14d33c2bc5a5
                    │                       └── link
                    └── _uploads
```

Registry的默认配置文件，位于实例所在容器的`/etc/docker/registry/config.yml`，默认基本内容是：
全量配置信息请见[config.yml](https://docs.docker.com/registry/configuration/#list-of-configuration-options)

``` console
~ # cat /etc/docker/registry/config.yml
version: 0.1
log:
  fields:
    service: registry
storage:
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
```

> Docker提供的标准镜像服务称为`registry`，当前版本是由go语言开发的v2（之前python开发的registry v1在网上已被标为废弃，最后版本是v1.6）, 新版 registry v2对镜像存储格式进行了重新设计，并且和旧版还不兼容，docker从1.6版本开始支持registry v2。

## 基于Registry的直通缓存服务（pull-through cache）

如果您的环境中运行着多个Docker实例，例如多个物理机或虚拟机都运行Docker，则每个Docker守护程序都将访问Internet，并从Docker Hub获取本地没有的映像。为此，我们可以部署一个本地Registry Mirror服务，并将所有Docker引擎的Mirror Sever指向该服务器，以免产生额外的互联网流量。

### 1. 服务端的部署方法

在一个已经安装Docker引擎的服务器`192.168.0.147`上部署一个Regsitry服务，其配置文件中，可以增加proxy配置段支持代理缓存服务，格式为：

``` yaml
proxy:
  remoteurl: https://registry-1.docker.io
  username: [username]
  password: [password]
```

> 获取Resistry默认配置文件的方法：
> `docker run -it --rm --entrypoint cat registry:2 /etc/docker/registry/config.yml`

因此，你可以在启动Registry服务时挂载自定义yml文件的方式，实现pull-through cache。

``` sh
docker run -d --restart=always --name registry \
    -p 5000:5000 \
    -v /data:/var/lib/registry \
    -v ./myconfig.yml:/var/lib/registry/config.yml \
    registry:2 
```

还有一个更方便的方法，不用修改配置文件，直接在启动Registry服务时设置环境变量即可，例如：

``` sh
docker run -p 5000:5000 -d --restart=always --name registry   \
  -e REGISTRY_PROXY_REMOTEURL=http://registry-1.docker.io \
  -v /data:/var/lib/registry \
  registry:2

```

### 2. 服务端的测试方法

检查Registry服务是否正常启动，可以通过以下方法：

```console
# curl http://localhost:5000/v2/_catalog
{"repositories":["myregistry"]}
```

### 3.客户端的配置方法

作为直通镜像使用方，需要修改Docker引擎配置文件`/etc/docker/daemon.json`，并在其中增加`registry-mirrors`信息段

``` sh
root@d7y-2:~# cat <<EOD >/etc/docker/daemon.json
{
"registry-mirrors": ["http://192.168.0.147:5000"]
}
EOD
```

当然，你也可以在Docker守护进程上，增加启动配置信息，例如：
`--registry-mirror=https://<my-docker-mirror-host>:<port-number>`

## 结论

1. Docker Registry提供的Pull-throgh cache服务部署很方便，但是缺乏UI管理界面导致维护比较困难，一般适合小规模本地部署。
2. 受到Docker pull机制的制约，Mirror仅支持Docker官方镜像，第三方或者私有仓库的mirror服务比较困难。
3. 由于Registry负责管理镜像的mainfest信息和镜像层文件，而所有docker pull都是基于上述标准流程和接口方式，因此Harbor、Dragnonfly等镜像管理产品都是必须基于Registry开发，并增加了一些复制、分发、UI管理等功能。

---

## 参考文献

- [Docker Mirror官方文档](https://docs.docker.com/registry/recipes/mirror/)
- [Harbor 2.1新增镜像代理和P2P镜像预热功能](https://www.sohu.com/a/421272143_609552)
- [为什么 Dragonfly 不能很好的支持 HTTPS 镜像仓库](https://github.com/dragonflyoss/Dragonfly/issues/525)
- [Github关于Add private-registry mirror support的讨论](https://github.com/moby/moby/pull/34319)
- [Docker Registry API接口示例](https://blog.csdn.net/ztsinghua/article/details/51496658)
- [容器OCI规范 镜像规范](https://blog.csdn.net/hyzhou33550336/article/details/65633502)
  