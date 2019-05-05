---
title: 为ECS设置ssh密钥登录的方法
date: 2019-05-04 20:11:55
tags:
---

最近，租用的腾讯云服务器ECS中招了，来自伦敦IDC的黑客通过ssh访问暴力破解root密码，并种上了木马（估计是挖矿的），看来必须重视安全加固工作了。 
ECS开放了80和443端口提供http和https服务，维护用的ssh端口22也是必须的。虽然可以为root设置更加复杂的密码，但也是给自己找麻烦，想来想去，最好还是取消ssh的密码登录方式，改为密钥登录方式，具体的操作步骤如下：

# 在腾讯云上，配置ECS采用ssh密钥登录方式

1. 登录腾讯云，并转到ECS控制台。**注意：为保障安全，ECS在关机状态才能修改登录方式！**

2. 选择`SSH密钥`-`创建密钥`，在弹出式窗口中选择`创建新密钥对`，并按提示进行操作。

> 弹窗中的`使用已有公钥`按钮，其功能是：导入用户自己手工产生的公钥文件

{% asset_img ecs-1.png %}

3. 根据提示信息下载私钥文件,注意自行妥善保存。默认文件后缀名为.dms，可以改为.txt。

4. 将新生成的密钥对与需要的ECS进行绑定。**绑定ECS成功后，ECS就不再支持密码登录了**

> 屏幕显示的公钥内容ssa-rsa，其实就是ECS服务器上`$HOME/.ssh/authorized_keys`的内容

{% asset_img ecs-2.png %}

# 在MAC上，用Terminal的命令行登录（需要指定密钥文件）

- Mac OS 用户请打开系统自带的终端（Terminal）并输入以下命令，**赋予私钥文件仅本人可读权限（必须设置！）**。

```bash
$ chmod 400 $HOME/Downloads/TENCENT_ECS.txt
```

- 在Terminal窗口中执行以下命令，进行远程登录。

```bash
$ ssh -i $HOME/Downloads/TENCENT_ECS.txt root@119.22.33.44
```

# 更简单的，Terminal无密码直接登录

在ssh指定密钥文件的基础上，还可以将ssh进一步简化为不需要指定密钥文件的信任方式，具体步骤如下：

1. 在mac终端生成公钥文件`id_rsa.pub`和私钥文件`id_rsa`。
```bash
$ ssh-keygen -t rsa
```

{% asset_img ecs-3.jpeg %}

> 图中红色标识的是是否对公钥和私钥进行对称加密，如果你输入了的话，那么在后续利用私钥登录的时候，需要输入该密码对私钥进行解密。如果安全性要求不高，可以不输入，直接点击enter跳过。

2. 在ECS服务器上寻找文件`$HOME/.ssh/authorized_keys`

3. 编辑该文件，添加`id_rsa.pub`里面的公钥数据。

4. 修改该文件的权限为644
```bash
$ chmod 644 $HOME/.ssh/authorized_keys
```

5. 现在MAC终端上可免密登录你的Linux服务器了，登录命令:
```bash
$ ssh root@119.22.33.44
```

> 实际上，你现在也可以在`Terminal`的菜单上直接选择`选择远程连接`，而不需要先打开本地的字符终端了

---
# 参考文档

- [腾讯云-添加密钥对的操作手册](https://cloud.tencent.com/document/product/213/16691#1.-创建密钥)
- [腾讯云-如何使用SSH密钥文件进行终端接入](https://cloud.tencent.com/document/product/213/5436#.E4.BD.BF.E7.94.A8-ssh-.E7.99.BB.E5.BD.95.EF.BC.88.E6.9C.AC.E5.9C.B0.E7.B3.BB.E7.BB.9F.E4.B8.BA-linux.2Fmac-os.EF.BC.89)