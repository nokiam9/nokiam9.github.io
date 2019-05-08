---
title: Kubernetes for Mac的安装方法和问题记录
date: 2019-04-01 16:21:24
tags:
---

## 安装Kubernetes的最简单方法
著名的Docker三剑客，Docker + Docker-compose + Docker Swarm。
docker-compose比较简单，适用于单机的服务编排，已经得到很好的应用，但是Docker Swarm作为集群的服务编排工具，受到K8s的强烈冲击，已经缴械投降了，目前Kubernets已经集成到了Docker中。
最简单的安装方法，就是在菜单Docker-Preferences-Kubernetes中，Enable即可。
{% asset_img k8s-install.png %}
如果看到‘Kubernetes is running...’，就是成功加载了！！！

## Dashboard的安装和启动
Kubernetes Dashboard是k8s集群的一个WEB UI管理工具。
代码托管在[https://github.com/kubernetes/dashboard](https://github.com/kubernetes/dashboard)

- 部署并启动Dashboard组件
```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

- 启动proxy代理服务，提供外部访问Kubernetes cluster
```bash
$ kubectl proxy
```

- 浏览器打开UI界面，URL地址是：
[http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/](http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/)

**注意：版本v1.8.3可以跳过认证方式**

## 如何创建Dashboard的认证鉴权文件（基于v1.10.1版本）

NOTE: apiVersion of ClusterRoleBinding resource may differ between Kubernetes versions. Prior to Kubernetes v1.8 the apiVersion was rbac.authorization.k8s.io/v1beta1.

- 创建权限文件`dashboard-adminuser.yaml`
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```

- 执行命令脚本，以创建admin-user用户角色，并获取token
```bash
$ kubectl apply -f dashboard-adminuser.yaml
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```
输出结果的示例：
```
Name:         admin-user-token-6gl6l
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=admin-user
              kubernetes.io/service-account.uid=b16afba9-dfec-11e7-bbb9-901b0e532516

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLTZnbDZsIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJiMTZhZmJhOS1kZmVjLTExZTctYmJiOS05MDFiMGU1MzI1MTYiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.M70CU3lbu3PP4OjhFms8PVL5pQKj-jj4RNSLA4YmQfTXpPUuxqXjiTf094_Rzr0fgN_IVX6gC4fiNUL5ynx9KU-lkPfk0HnX8scxfJNzypL039mpGt0bbe1IXKSIRaq_9VW59Xz-yBUhycYcKPO9RM2Qa1Ax29nqNVko4vLn1_1wPqJ6XSq3GYI8anTzV8Fku4jasUwjrws6Cn6_sPEGmL54sq5R4Z5afUtv-mItTmqZZdxnkRqcJLlg2Y8WbCPogErbsaCDJoABQ7ppaqHetwfM_0yMun6ABOQbIwwl8pspJhpplKwyo700OSpvTT9zlBsu-b35lzXGBRHzv5g_RA
```

- 在Dashboard的UI登陆界面中，粘贴`token`的数据字段，
{% asset_img dash-login.png %}

- 成功展示UI！！！
{% asset_img dash-main.png %}

---
## 问题1：激活Kubernetes时，一直停留在‘Kubernetes is starting...’
- 根本原因：激活k8s需要自动下载google的镜像文件，但一直无法通过GW
- 解决方法：在docker-Preferences-Proxies-Manunal Proxy Configration中，设置代理服务器为`host.docker.internal:1087`，由于国际传输速度较慢，可能需要几个小时
- 补充说明：由于docker是运行在Mac的虚拟机上，无法直接使用Shadowsocks的默认代理设置`http://127.0.0.1:1087`，需要将localhost地址替换为Mac网卡的物理地址，最佳建议是使用docker的内部DNS域名`host.docker.internal:1087`，以避免WIFI切换时修改物理地址

## 问题2：