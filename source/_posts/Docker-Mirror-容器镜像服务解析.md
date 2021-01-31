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

> image已经建立了行业标准 - OCI(Open Container Initiative)镜像规范，确保Docker、Podman等不同容器产品之间共享image，可以参见附录继续研究

docker pull的过程很复杂，包括鉴权、校验，下载、合并镜像层，解压缩等等，但最核心的是两个步骤：

1. 发送请求 `GET /v2/<name>/manifests/<reference>` ，获取镜像的mainfest清单文件
    reference可以是标记tag，或摘要digest。
2. 发送请求 `GET /v2/<name>/blobs/<digest>` ，获取镜像层文件

通过`curl -X GET http://localhost:5000/v2/alpine/manifests/3.6`，获取镜像的manifest清单文件，其格式为

``` json
{
   "schemaVersion": 1,
   "name": "alpine",
   "tag": "3.6",
   "architecture": "amd64",
   "fsLayers": [
      {
         "blobSum": "sha256:a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4"
      },
      {
         "blobSum": "sha256:5a3ea8efae5d0abb93d2a04be0a4870087042b8ecab8001f613cdc2a9440616a"
      }
   ],
   "history": [
      {
         "v1Compatibility": "{\"architecture\":\"amd64\",\"config\":{\"Hostname\":\"\",\"Domainname\":\"\",\"User\":\"\",\"AttachStdin\":false,\"AttachStdout\":false,\"AttachStderr\":false,\"Tty\":false,\"OpenStdin\":false,\"StdinOnce\":false,\"Env\":[\"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin\"],\"Cmd\":[\"/bin/sh\"],\"ArgsEscaped\":true,\"Image\":\"sha256:143f9315f5a85306192ccffd37fbfa65db21f67aaa938c2538bd50f52123a12f\",\"Volumes\":null,\"WorkingDir\":\"\",\"Entrypoint\":null,\"OnBuild\":null,\"Labels\":null},\"container\":\"fd086f4b9352674c6a1ae4d02051f95a4e0a55cda943c5780483938dedfb2d8f\",\"container_config\":{\"Hostname\":\"fd086f4b9352\",\"Domainname\":\"\",\"User\":\"\",\"AttachStdin\":false,\"AttachStdout\":false,\"AttachStderr\":false,\"Tty\":false,\"OpenStdin\":false,\"StdinOnce\":false,\"Env\":[\"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin\"],\"Cmd\":[\"/bin/sh\",\"-c\",\"#(nop) \",\"CMD [\\\"/bin/sh\\\"]\"],\"ArgsEscaped\":true,\"Image\":\"sha256:143f9315f5a85306192ccffd37fbfa65db21f67aaa938c2538bd50f52123a12f\",\"Volumes\":null,\"WorkingDir\":\"\",\"Entrypoint\":null,\"OnBuild\":null,\"Labels\":{}},\"created\":\"2019-03-07T22:20:00.563496859Z\",\"docker_version\":\"18.06.1-ce\",\"id\":\"baaf9c1caf4fb211f173d053029997dcfade0644ac354c8a068e4ebf23fcf1c5\",\"os\":\"linux\",\"parent\":\"5d8f720b0ab2b92a29a7e338aa90cad32dac2bf6518c7aae5844aab896ee36ec\",\"throwaway\":true}"
      },
      {
         "v1Compatibility": "{\"id\":\"5d8f720b0ab2b92a29a7e338aa90cad32dac2bf6518c7aae5844aab896ee36ec\",\"created\":\"2019-03-07T22:20:00.434038891Z\",\"container_config\":{\"Cmd\":[\"/bin/sh -c #(nop) ADD file:9714761bb81de664e431dec41f12db20f0438047615df2ecd9fdc88933d6c20f in / \"]}}"
      }
   ],
   "signatures": [
      {
         "header": {
            "jwk": {
               "crv": "P-256",
               "kid": "KMUD:PVGZ:P2LO:LC7C:JSZZ:DCUO:FB3A:VXIA:U7UM:WVMY:7KJT:5TPS",
               "kty": "EC",
               "x": "M5uDyG_04QhZKJx2FFku4t2UUWeYYeyg0-LhmrNf5OQ",
               "y": "TdXi_yetmEqWSEKqTudnjb7tn5m-AVQmxGeknXVL8w8"
            },
            "alg": "ES256"
         },
         "signature": "xSwT9ePqDKjDm3i9AHkgFxnZGO6TdVIePcl6XxTvrCPSjOx_Xd1jf8YgouhXWDffBygicwp8DDnxJ7bB30puuw",
         "protected": "eyJmb3JtYXRMZW5ndGgiOjIxMzAsImZvcm1hdFRhaWwiOiJDbjAiLCJ0aW1lIjoiMjAyMS0wMS0zMVQxMzo1NjoxNFoifQ"
      }
   ]
}
```

## Docker Registry 镜像注册服务

Registry是Docker提供的镜像注册服务，其设计目标是一套存储和分发镜像的处理机制，实现方式是在Docker引擎中运行一个来自`docker.io/library/registry`的实例。

需要注意的是，Registry管理镜像是基于文件系统的,数据存储目录位于实例所在容器的`/var/lib/registry`

``` console
~ # tree /var/lib/registry
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
            │       ├── 5a
            │       │   └── 5a3ea8efae5d0abb93d2a04be0a4870087042b8ecab8001f613cdc2a9440616a
            │       │       └── data
            │       └── a3
            │           └── a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4
            │               └── data
            └── repositories
                ├── alpine
                │   ├── _layers
                │   │   └── sha256
                │   │       ├── 43773d1dba76c4d537b494a8454558a41729b92aa2ad0feb23521c3e58cd0440
                │   │       │   └── link
                │   │       ├── 5a3ea8efae5d0abb93d2a04be0a4870087042b8ecab8001f613cdc2a9440616a
                │   │       │   └── link
                │   │       └── a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4
                │   │           └── link
                │   ├── _manifests
                │   │   ├── revisions
                │   │   │   └── sha256
                │   │   │       └── 36c3a913e62f77a82582eb7ce30d255f805c3d1e11d58e1f805e14d33c2bc5a5
                │   │   │           └── link
                │   │   └── tags
                │   │       └── 3.6
                │   │           ├── current
                │   │           │   └── link
                │   │           └── index
                │   │               └── sha256
                │   │                   └── 36c3a913e62f77a82582eb7ce30d255f805c3d1e11d58e1f805e14d33c2bc5a5
                │   │                       └── link
                │   └── _uploads
                └── myregistry
                    ├── _layers
                    │   └── sha256
                    │       ├── 43773d1dba76c4d537b494a8454558a41729b92aa2ad0feb23521c3e58cd0440
                    │       │   └── link
                    │       ├── 5a3ea8efae5d0abb93d2a04be0a4870087042b8ecab8001f613cdc2a9440616a
                    │       │   └── link
                    │       └── a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4
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

48 directories, 16 files
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

如果您的环境中运行着多个Docker实例，例如多个物理机或虚拟机都运行Docker，则每个Docker守护程序都将访问Internet，并从Docker Hub获取本地没有的映像。
为此，我们可以部署一个本地Registry Mirror服务，并将所有Docker引擎的Mirror Sever指向该服务器，以免产生额外的互联网流量。

### Registry mirror的原理

Docker Hub的镜像数据分为两部分：index数据和registry数据。前者保存了镜像的一些元数据信息，数据量很小；后者保存了镜像的实际数据，数据量比较大。平时我们使用docker pull命令拉取一个镜像时的过程是：先去index获取镜像的一些元数据，然后再去registry获取镜像数据。

所谓registry mirror就是搭建一个registry，然后将docker hub的registry数据缓存到自己本地的registry。
整个过程是：当我们使用docker pull去拉镜像的时候，会先从我们本地的registry mirror去获取镜像数据，如果不存在，registry mirror会先从docker hub的registry拉取数据进行缓存，再传给我们。

整个过程是流式的，registry mirror并不会等全部缓存完再给我们传，而且边缓存边给客户端传。

对于缓存，我们都知道一致性非常重要。
registry mirror与docker官方保持一致的方法是：registry mirror只是缓存了docker hub的registry数据，并不缓存index数据。
所以我们pull镜像的步骤是：

- 先连docker hub的index获取镜像的元数据，如果我们registry mirror里面有该镜像的缓存，且数据与从index处获取到的元数据一致，则从registry mirror拉取；
- 如果我们的registry mirror有该镜像的缓存，但数据与index处获取的元数据不一致，或者根本就没有该镜像的缓存，则先从docker hub的registry缓存或者更新数据。

### 1. 服务端的部署方法

在一个已经安装Docker引擎的服务器`192.168.0.147`上部署一个Regsitry服务，其配置文件中，可以增加proxy配置段支持代理缓存服务，格式为：

``` yaml
proxy:
  remoteurl: https://registry-1.docker.io
  username: [username]
  password: [password]
```

注意，缓存默认保存7天，配置信息位于：

``` yaml
storage:
  maintenance:
    uploadpurging:
      enabled: true
a      age: 168h
      interval: 24h
      dryrun: false
    readonly:
      enabled: false
```

`uploadpurging`是一个后台过程，该过程会定期从注册表的`uploads`目录中删除孤立的文件。
默认情况下启用该后台进程，并设置以下参数。

> TODO: 缓存的镜像的有效期默认是一周（168hour），这个配置似乎并不解决？？？

|参数名称|必选项|描述|
|:---:|:---:|:---|
|enabled|yes|Set to true to enable upload purging. Defaults to true.|
|age|yes|Upload directories which are older than this age will be deleted.Defaults to 168h (1 week).|
|interval|yes|The interval between upload directory purging. Defaults to 24h.|
|dryrun|yes|Set dryrun to true to obtain a summary of what directories will be deleted. Defaults to false.|

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
  -e REGISTRY_PROXY_REMOTEURL=https://registry-1.docker.io \
  -v /data:/var/lib/registry \
  registry:2

```

### 2. 服务端的测试方法

检查Registry服务是否正常启动，可以通过以下方法：

```console
[root@bl-local ~]# curl -ski -X GET http://localhost:5000/v2/_catalog
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Docker-Distribution-Api-Version: registry/2.0
X-Content-Type-Options: nosniff
Date: Sun, 31 Jan 2021 15:58:44 GMT
Content-Length: 66

{"repositories":["calico/cni","library/alpine","library/python"]}
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
- [搭建registry mirror的操作记录](https://niyanchun.com/deploy-registry-mirror.html)
- [一种Harbor部署私有Mirror服务的不正规方法](https://cloud.tencent.com/developer/article/1413239)
- [Harbor 2.1新增镜像代理和P2P镜像预热功能](https://www.sohu.com/a/421272143_609552)
- [为什么 Dragonfly 不能很好的支持 HTTPS 镜像仓库](https://github.com/dragonflyoss/Dragonfly/issues/525)
- [Github关于Add private-registry mirror support的讨论](https://github.com/moby/moby/pull/34319)
- [Docker Registry API接口示例](https://blog.csdn.net/ztsinghua/article/details/51496658)
- [容器OCI规范 镜像规范](https://blog.csdn.net/hyzhou33550336/article/details/65633502)
- [开放容器标准(OCI) 内部分享](https://xuanwo.io/2019/08/06/oci-intro/)
- [浙江移动容器云基于 Dragonfly 的统一文件分发平台生产实践](https://developer.aliyun.com/article/707053)
