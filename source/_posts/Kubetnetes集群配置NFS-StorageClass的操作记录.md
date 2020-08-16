---
title: Kubetnetes集群配置NFS StorageClass的操作记录
date: 2020-08-16 11:44:35
tags:
---

## 关于Kubernetes Volume

Kubernetes 和 Docker 类似，也是通过 Volume 的方式提供对存储的支持，Volume 被定义在 Pod 上，可以被 Pod 里的多个容器挂载到相同或不同的路径下。
但是，Kubernetes 中 Volume 的 概念与Docker 中的 Volume 也存在重大区别，具体区别如下：

- Kubernetes 中的 Volume 与 Pod 的生命周期相同，但与容器的生命周期不相关。当容器终止或重启时，Volume 中的数据也不会丢失。
- 当 Pod 被删除时，Volume 才会被清理。并且数据是否丢失取决于 Volume 的具体类型，比如：emptyDir 类型的 Volume 数据会丢失，而 PV 类型的数据则不会丢失。

Volume 的核心是目录，可以通过 Pod 中的容器来访问。该目录是如何形成的、支持该目录的介质以及其内容取决于所使用的特定卷类型。要使用 Volume，需要为 Pod 指定为 Volume (`spec.volumes` 字段) 以及将它挂载到容器的位置 (`spec.containers.volumeMounts` 字段)。Kubernetes 支持多种类型的卷，一个 Pod 可以同时使用多种类型的 Volume。

容器中的进程看到的是由其 Docker 镜像和 Volume 组成的文件系统视图。 Docker 镜像位于文件系统层次结构的根目录，任何 Volume 都被挂载在镜像的指定路径中。Volume 无法挂载到其他 Volume 上或与其他 Volume 的硬连接。Pod 中的每个容器都必须独立指定每个 Volume 的挂载位置。

Kubernetes 目前支持多种 Volume 类型，大致如下：

- `emptryDir`: 顾名思义是一个空目录，它的生命周期和所属的 Pod 是完全一致的，主要用于某些应用程序无需永久保存的临时目录，在多个容器之间共享数据等。
  缺省情况下，emptryDir 是使用主机磁盘进行存储的。你也可以使用其它介质作为存储，比如：网络存储、内存等。设置 `emptyDir.medium` 字段的值为 Memory 就可以使用内存进行存储，使用内存做为存储可以提高整体速度，但是要注意一旦机器重启，内容就会被清空，并且也会受到容器内存的限制。

- `hostPath`: hostPath 类型的 Volume 允许用户挂载 Node 宿主机上的文件或目录到 Pod 中。大多数 Pod 都用不到这种 Volume，其缺点比较明显，比如：Pod 在不同节点上的行为可能会有所不同；在底层主机上创建的文件或目录只能由 root 写入，您需要在特权容器中以 root 身份运行进程等。

    这种类型的 Volume 主要用在以下场景中：
  - 运行中的容器需要访问 Docker 内部的容器，使用 /var/lib/docker 来做为 hostPath 让容器内应用可以直接访问 Docker 的文件系统。在容器中运行 cAdvisor，使用 /dev/cgroups 来做为 hostPath。
  - 和 DaemonSet 搭配使用，用来操作主机文件。例如：日志采集方案 FLK 中的 FluentD 就采用这种方式来加载主机的容器日志目录，达到收集本主机所有日志的目的。

- `secret`： secret volume用于将敏感信息（如密码）传递给pod。可以将secrets存储在Kubernetes API中，使用的时候以文件的形式挂载到pod中，而不用连接api。 secret volume由tmpfs（RAM支持的文件系统）支持。

- `persistentVolumeClaim`:  persistentVolumeClaim用来挂载持久化磁盘的。PersistentVolumes是用户在不知道特定云环境的细节的情况下，实现持久化存储（如GCE PersistentDisk或iSCSI卷）的一种方式。这是最主要的存储持久化方案，也是本文讨论的重点

当然，Kubernetes还支持cephfs、glusterfs等分布式存储，和awsElasticBlockStore、azureFileVolume等云计算厂商的存储方案，以及Flocker等开源的容器集群数据卷管理器等。

## 关于PV、PVC和SC

先介绍Kubernetes存储管理的几个核心概念：

- 持久化卷（PV - PersistentVolume)：负责提供网络存储资源，是对底层的共享存储的一种抽象，PV 由管理员进行创建和配置，它和具体的底层的共享存储技术的实现方式有关，比如 Ceph、GlusterFS、NFS 等，都是通过插件机制完成与共享存储的对接。
- 持久化卷声明（PVC - PersistentVolumeClaim)：负责为POD请求存储资源。PVC 是用户存储的一种声明，PVC 和 Pod 比较类似，Pod 消耗的是节点，PVC 消耗的是 PV 资源，Pod 可以请求 CPU 和内存，而 PVC 可以请求特定的存储空间和访问模式。对于真正使用存储的用户不需要关心底层的存储实现细节，只需要直接使用 PVC 即可。
- 存储类（SC - StorageClass）：负责动态创建 PV，可以封装不同类型的存储供 PVC 选用。通过 StorageClass 的定义，管理员可以将存储资源定义为某种类型的资源，比如快速存储、慢速存储等，用户根据 StorageClass 的描述就可以非常直观的知道各种存储资源的具体特性了，这样就可以根据应用的特性去申请合适的存储资源了。

StorageClass 包括四个部分:

- provisioner：指定 Volume 插件的类型，包括内置插件（如 kubernetes.io/glusterfs）和外部插件（如 external-storage 提供的 ceph.com/cephfs）。
- mountOptions：指定挂载选项，当 PV 不支持指定的选项时会直接失败。比如 NFS 支持 `hard` 和 `nfsvers=4.1`等选项。
- parameters：指定 provisioner 的选项，比如 kubernetes.io/aws-ebs 支持 type、zone、iopsPerGB 等参数。
- reclaimPolicy：指定回收策略，同 PV 的回收策略。

## 配置步骤

### 1. 创建NFS服务

NFS SERVER的安装方法参见[安装NFS服务](https://developer.aliyun.com/article/610391)
Client的配置参数为：

- 服务地址：    `192.168.0.200`
- 共享数据目录： `/data`

### 2. 部署存储供应卷

实际上就是部署一个运行NFS Client的Container，其任务是使用我们已经配置好的 nfs 服务器，并根据PVC的请求, 动态创建持久卷，也就是自动帮我们创建PV。

- 该容器的名称为`nfs-client-provisioner`
- 基础镜像来自于`quay.io/external_storage/nfs-client-provisioner:latest`，当前版本号为`v3.1.0-k8s1.11`
- 供应者(provisioner)的名称为`fuseim.pri/ifs`

``` bash

cat > nfs-client.yaml << EOF
kind: Deployment
apiVersion: apps/v1
metadata:
name: nfs-client-provisioner
spec:
replicas: 1
strategy:
    type: Recreate
selector:
    matchLabels:
    app: nfs-client-provisioner
template:
    metadata:
    labels:
        app: nfs-client-provisioner
    spec:
    serviceAccountName: nfs-client-provisioner
    containers:
        - name: nfs-client-provisioner
        image: quay.io/external_storage/nfs-client-provisioner:latest
        volumeMounts:
            - name: nfs-client-root
            mountPath: /persistentvolumes
        env:
            - name: PROVISIONER_NAME
            value: fuseim.pri/ifs
            - name: NFS_SERVER
            value: 192.168.0.200        # < Your NFS Server IP >
            - name: NFS_PATH
            value: /data                # < Your NFS Server MountDir >
    volumes:
        - name: nfs-client-root
        nfs:
            server: 192.168.0.200       # < Your NFS Server IP >
            path: /data                 # < Your NFS Server MountDir >
EOF
```

### 3. 为存储供应卷创建并绑定权限规则集

``` sh
cat > nfs-client-sa.yaml << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
name: nfs-client-provisioner

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
name: nfs-client-provisioner-runner
rules:
- apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
- apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
- apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
- apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
- apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
name: run-nfs-client-provisioner
subjects:
- kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
kind: ClusterRole
name: nfs-client-provisioner-runner
apiGroup: rbac.authorization.k8s.io
EOF
```

### 4. 部署storageclass

定义StorageClass的名称为`managed-nfs-storage`，并从名为`fuseim.pri/ifs` 的provisioner获得动态PV资源，

``` sh
cat > nfs-client-class.yaml << EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
    name: managed-nfs-storage
provisioner: fuseim.pri/ifs     # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
    archiveOnDelete: "false"    # When set to "false" your PVs will not be archived
                                # by the provisioner upon deletion of the PVC.
EOF
```

### 5. 运行上述三个部署文件

``` sh
kubectl create -f nfs-client.yaml
kubectl create -f nfs-client-sa.yaml
kubectl create -f nfs-client-class.yaml
```

到此为止，大功告成！！！

## 注意事项

### 1. 将自定义的StorageClass设置为Kubernetes集群的默认StorageClass

在使用 PVC 时，可以通过`DefaultStorageClass`准入控制设置默认 StorageClass, 即给未设置 storageClassName 的 PVC 自动添加默认的 StorageClass。而默认的 StorageClass 带有 `storageclass.kubernetes.io/is-default-class=true`的 annotation 。

设置为默认 StorageClass

``` sh
kubectl patch storageclass managed-nfs-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

取消原来的默认 StorageClass

``` sh
kubectl patch storageclass managed-nfs-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

### 2. 常用的检查方法

``` sh
[root@Helm harbor]# kubectl get sc
NAME                 PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
course-nfs-storage   fuseim.pri/ifs   Delete          Immediate           false                  13h

[root@Helm harbor]# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                         STORAGECLASS         REASON   AGE
pvc-2cbc8907-acbd-4999-b5ea-c58000347835   1Gi        RWO            Delete           Bound    default/data-ggg-harbor-redis-0               course-nfs-storage            12h
pvc-3e1e2ac4-04dc-44a6-9f6e-7456f0d09d5c   5Gi        RWO            Delete           Bound    default/ggg-harbor-registry                   course-nfs-storage            12h
pvc-535149b4-b3fb-43cb-b8e3-9d7a1069e7b9   5Gi        RWO            Delete           Bound    default/data-ggg-harbor-trivy-0               course-nfs-storage            12h
pvc-717e79cd-3c44-44a0-8b2b-ce72b1ae0994   5Gi        RWO            Delete           Bound    default/ggg-harbor-chartmuseum                course-nfs-storage            12h
pvc-7a76ad65-dcbc-4310-ab66-eaf658c20b2f   1Gi        RWO            Delete           Bound    default/ggg-harbor-jobservice                 course-nfs-storage            12h
pvc-f9615144-0a24-4fdf-b12f-fe5d4ff5aed3   1Gi        RWO            Delete           Bound    default/database-data-ggg-harbor-database-0   course-nfs-storage            12h

[root@Helm harbor]# kubectl get pvc
NAME                                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS         AGE
data-ggg-harbor-redis-0               Bound    pvc-2cbc8907-acbd-4999-b5ea-c58000347835   1Gi        RWO            course-nfs-storage   12h
data-ggg-harbor-trivy-0               Bound    pvc-535149b4-b3fb-43cb-b8e3-9d7a1069e7b9   5Gi        RWO            course-nfs-storage   12h
database-data-ggg-harbor-database-0   Bound    pvc-f9615144-0a24-4fdf-b12f-fe5d4ff5aed3   1Gi        RWO            course-nfs-storage   12h
ggg-harbor-chartmuseum                Bound    pvc-717e79cd-3c44-44a0-8b2b-ce72b1ae0994   5Gi        RWO            course-nfs-storage   12h
ggg-harbor-jobservice                 Bound    pvc-7a76ad65-dcbc-4310-ab66-eaf658c20b2f   1Gi        RWO            course-nfs-storage   12h
ggg-harbor-registry                   Bound    pvc-3e1e2ac4-04dc-44a6-9f6e-7456f0d09d5c   5Gi        RWO            course-nfs-storage   12h

[root@Helm harbor]# kubectl get pods
NAME                                        READY   STATUS    RESTARTS   AGE
ggg-harbor-chartmuseum-7d8ccfc449-85lrh     1/1     Running   0          12h
ggg-harbor-clair-bfd9dc556-sdc6t            2/2     Running   2          12h
ggg-harbor-core-6bbb76fb97-2ztrf            1/1     Running   2          12h
ggg-harbor-database-0                       1/1     Running   0          12h
ggg-harbor-jobservice-59c9b578d6-5k8l4      1/1     Running   1          12h
ggg-harbor-nginx-76b69b5fd8-swmb6           1/1     Running   0          12h
ggg-harbor-notary-server-7899447446-ns7h8   1/1     Running   4          12h
ggg-harbor-notary-signer-666cdc5478-wgx62   1/1     Running   4          12h
ggg-harbor-portal-6d465b4f77-67d5v          1/1     Running   0          12h
ggg-harbor-redis-0                          1/1     Running   0          12h
ggg-harbor-registry-6847fbfc64-klpvd        2/2     Running   0          12h
ggg-harbor-trivy-0                          1/1     Running   1          12h
nfs-client-provisioner-7895fccfdc-p8tw9     1/1     Running   0          13h

```

### 3. 动态创建PV的存储方式

自动创建的 PV 以`${namespace}-${pvcName}-${pvName}`这样的命名格式创建在 NFS 服务器上的共享数据目录中
而当这个 PV 被回收后会以`archieved-${namespace}-${pvcName}-${pvName}`这样的命名格式存在 NFS 服务器上。

``` sh
[root@nfs data]# ls -l |grep ggg
drwx------. 19 polkitd input  4096 8月  16 01:02 default-database-data-ggg-harbor-database-0-pvc-f9615144-0a24-4fdf-b12f-fe5d4ff5aed3
drwxrwxrwx.  2 root    root   4096 8月  16 13:12 default-data-ggg-harbor-redis-0-pvc-2cbc8907-acbd-4999-b5ea-c58000347835
drwxrwxrwx.  4 root    root   4096 8月  16 01:00 default-data-ggg-harbor-trivy-0-pvc-535149b4-b3fb-43cb-b8e3-9d7a1069e7b9
drwxrwxrwx.  2 root    root   4096 8月  16 01:00 default-ggg-harbor-chartmuseum-pvc-717e79cd-3c44-44a0-8b2b-ce72b1ae0994
drwxrwxrwx.  2 root    root   4096 8月  16 01:00 default-ggg-harbor-jobservice-pvc-7a76ad65-dcbc-4310-ab66-eaf658c20b2f
drwxrwxrwx.  2 root    root   4096 8月  16 01:00 default-ggg-harbor-registry-pvc-3e1e2ac4-04dc-44a6-9f6e-7456f0d09d5c

[root@nfs data]# ls -l |grep kkk
drwx------. 19 polkitd input  4096 8月  16 00:38 archived-default-database-data-kkk-harbor-database-0-pvc-4e9b9e6c-3729-464d-9b9f-57f916533b8f
drwxrwxrwx.  2 root    root   4096 8月  16 00:47 archived-default-data-kkk-harbor-redis-0-pvc-208f1203-f5fe-4a8c-b8cd-c8734fecdd76
drwxrwxrwx.  4 root    root   4096 8月  16 00:23 archived-default-data-kkk-harbor-trivy-0-pvc-e93d5ae0-d84f-440d-b1bc-ead19485fa00
drwxrwxrwx.  2 root    root   4096 8月  16 00:23 archived-default-kkk-harbor-chartmuseum-pvc-49dcb036-30d2-453e-b3fc-851518c27b9b
drwxrwxrwx.  2 root    root   4096 8月  16 00:23 archived-default-kkk-harbor-jobservice-pvc-19df4cf0-3f04-439d-9b72-42377834b514
drwxrwxrwx.  2 root    root   4096 8月  16 00:23 archived-default-kkk-harbor-registry-pvc-11d4f04e-cb6b-4d36-90c0-99ae85f5d10c
```

### 4. 通过HELM部署nfs-client

如果能够KX上网，从HELM直接安装NFS StorageClass也是很方便的。

`$ helm install hhh stable/nfs-client-provisioner --set nfs.server=192.168.0.200 --set nfs.path=/data`

安装完成后将部署一个名为`nfs-client`的StorageClass，其配置信息为：

``` yaml
---
allowVolumeExpansion: true
metadata:
  annotations:
    meta.helm.sh/release-name: hhh                          # HELM的Release Name，这个不重要
    meta.helm.sh/release-namespace: default
  creationTimestamp: '2020-08-16T05:49:31Z'
  labels:
    app: nfs-client-provisioner
    app.kubernetes.io/managed-by: Helm
chart: nfs-client-provisioner-1.2.9                         # HELM的配置文件版本号
    heritage: Helm
    release: hhh
  managedFields:
    - apiVersion: storage.k8s.io/v1
      fieldsType: FieldsV1
      fieldsV1:
        'f:metadata': {}
      manager: Go-http-client
      operation: Update
      time: '2020-08-16T05:49:31Z'
  name: nfs-client                                          # 新建StorageClass的Name
  resourceVersion: '139695'
  selfLink: /apis/storage.k8s.io/v1/storageclasses/nfs-client
  uid: d7295404-a65e-4b56-8bae-bc018b85d032
parameters:
  archiveOnDelete: 'true'
provisioner: cluster.local/hhh-nfs-client-provisioner       # provisioner的container名字不同
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

---

## 参考资料

- [Kubernetes 学习手册-阳明博客](https://www.qikqiak.com/k8s-book/docs/35.StorageClass.html)
- [Kubernetes NFS-Client Provisioner 官方文档](https://github.com/kubernetes-retired/external-storage/tree/master/nfs-client)
- [浅谈 Kubernetes 数据持久化方案](https://www.sohu.com/a/249429452_760387)
- [通过HELM安装nfs-client-provisioner](https://github.com/helm/charts/tree/master/stable/nfs-client-provisioner)
- [Kubernetes Volume综述](http://docs.kubernetes.org.cn/429.html)