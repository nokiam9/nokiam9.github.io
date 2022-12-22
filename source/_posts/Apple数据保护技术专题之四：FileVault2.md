---
title: Apple数据保护技术专题之四：FileVault2
date: 2022-12-08 11:19:15
tags:
---

从本质上说，FileVault 属于 FDE（Full Disk Encryption，全盘加密）技术，早期的 Android 设备也是采用此方案，但随着技术发展，Andriod 和 iOS 逐渐都演进为 FBE（File Based Encryption，文件加密），仅在 MacOS 中保留了 FileVault2，但同时也支持 DataProtection 数据保护技术。

## 一、Legacy FileVault

2003年，Mac OS X Panther (黑豹，10.3) 首次发布了 FileVault，当时的版本只加密了用户目录（而非启动宗卷），方法是采用一个巨大的稀疏磁盘镜像文件（sparse disk image）作为虚拟磁盘，负责存储加密的文件元数据。在启用 FileVault 功能时，系统会要求用户设置一个**主密码**，当用户忘记密码时，需要这个主密码或是恢复密钥直接解密文件。
> 与磁盘镜像文件`.dmg`不同，稀疏磁盘镜像文件`.sparseimage`的实际磁盘占用空间是根据使用情况动态扩展的

2006年，Mac OS X Leopard（花豹，10.5）做了少许改进，将虚拟磁盘改为稀疏绑定磁盘镜像（sparse bundle disk image），特点是将文件分割为若干个 8MB 的小文件（Band），在文件修改时无需复制全部内容，仅复制本次修改的 Band，因此备份速度更快，非常适合 Time Machine 的需求。

Apple 将上述两种版本的技术统称为legacy FileVault，其存在很多严重问题，包括：

- 只加密了用户目录的元数据，实际上数据文件的内容并未加密
- 采用 1024-bit RSA 或 3DES-EDE 算法已被证明存在破解风险，加密算法 CBC 模式的安全强度不足
- 当 MacOS 置于睡眠模式时，密钥存在的泄露风险

### 二、FileVault 2

2010年，Mac OS X Lion (狮子，10.7) 发布了重新设计的 FDE 方案 - FileVault 2，其技术特点包括：

1. 启动卷宗的分区由 HFS+ 文件系统改为一个 CoreStorage 管理的加密卷，并增加了一个 Recovery HD 分区卷
2. 使用**用户登录密码**作为加密令牌，
3. 加密标准采用 NIST 推荐的 AES-128-AES 模式，分为并采用AES的XTS-AES模式将数据划分成128位的块，同时生成一枚256位密钥来加密磁盘，这一标准也是NIST推荐的标准。当用户解锁后，其他用户也可以访问这些数据，直到电脑关机。

> CBC 引入了一个IV值递归　上一组加密后产生新的IV值作为下一数据块加密的输入 这样至少有个一个参数不同　但带来的问题是没有办法随机访问各个区块了，访问某个区块必须解出前一个区块数据。
> XTS 模式加入了 tweak key 与 AES key 互相配合，各个区块用同样的AES key 但Tweak key 不相同 比如设置为与区块的index 成对应关系, 这样各个区块数据加密解密互相独立（独立的tweak key)
> 
抛弃了基于 HFS+ 文件系统，通过磁盘镜像对 HFS+ 文件系统的元数据加密的技术路线，采用了全新设计的




- 直接加密整个 Mac OS 的启动宗卷（自然也包括用户目录），并
- 采用这种加密方法，已授权的用户信息将从一个单独的非加密引导卷宗读取。(分区格式为 Apple_Boot)。

FileVault

约有3%的I/O的性能下降会出现在使用AES指令集的CPU上（如Intel Broadwell 架构CPU， OS X 10.10.3环境），早期的酷睿CPU，或者其他不使用该指令集的处理器会有更明显的性能下降。

系统运行时执行 FileVault2，电脑会创建一个**恢复密钥**，并在屏幕上显示出来，提示用户保管，并且提供了一个将密钥上传至 Apple 的可选项。恢复密钥共有120位，由全部英文字母和数字 1-9 组成，系统会调用/dev/random的随机数生成器生成整枚密钥。因此其安全性取决于macOS 使用的随机数生成算法，根据密码学家 2012 年的声明指出，该算法是安全的。

除非重新加密整个卷宗，否则将无法修改恢复密钥。

![Volume Main Key](vmk.png)

在了解了 FileVault2 加密的流程之后，可以根据具体方 法得到相对应的解密流程。将加密后的磁盘用外部工具加 载观察，从磁盘结构来看，加密后的硬盘与加密前硬盘发 生了重大的变化。通过对加密数据研究和相关资料文献的阅读，得 知加密卷的卷头存储了解密所需的各种相关参数 [4]。图 2 为 FileVault2 解密流程。

根据对加密卷数据分析可得加密卷的头部主要包含有 如下信息 :
卷头部签名、块大小、卷大小、元数据大小、第一个 元数据块块号、第二个元数据块块号、第三个元数据块块号、 第四个元数据块块号、加密方法、逻辑卷 ID(用于解密加 密的密钥文件)、物理卷 ID[5]。
解密步骤如下 :
1)识别到 FileVault2 加密卷和存储的需要提取的用户 密码生成的主密钥信息或者恢复令牌的 EncryptedRoot.plist. wipekey 文件 ;
2)解密通过文件系统解析获得的 EncryptedRoot.plist. wipekey 文件后，提取存储的密钥信息 ;
3)输入用户登录密码或者恢复密钥，验证步骤 2)中 提取到的密钥信息是否正确 ;
4)验证正确后根据用户密码或者恢复密钥及加密的 EncryptedRoot.plist.wipekey 文件中存储的密钥信息生成主 密钥和 tweak key ;
5)用生成的两个解密密钥信息(key1 和 key2)及加密 卷数据进行解密，生成解密数据即完成解密。
用取证工具或其他可以解析 HFSPlus 文件系统的工具 对解密数据进行解析，可以验证解密结果是否正确。


![密钥层次架构](arch.png)


### VEK - Volume Encryption Key，卷宗封装密钥

---

关键文件：EncryptedRoot.plist

Once decrypted, the file EncryptedRoot.plist has an XML structure with the following important entries:
• PassphraseWrappedKEKStruct(1)
– 284 byte structure for recovery password.
• PassphraseWrappedKEKStruct(2)
– 284 byte structure for user password.
• KEKWrappedVolumeKeyStruct(1) – unused.
• KEKWrappedVolumeKeyStruct(2)
– contains wrapped volume master key.


``` c
p = get_user_password()                                 // 用户输入password 
salt = get_salt_from_PassphraseWrappedKEK()
iterations = 41000
pk = pbkdf2(p, salt, iterations, HMAC-SHA256)           // 计算PDK
kek_wrapped = get_kek_from_PassphraseWrappedKEK()           
kek = aes_unwrap(kek_wrapped, pk)                       // 获得
vmk_wrapped = get_vmk_from_KEKWrappedVolumeKey()
vmk = aes_unwrap(vmk_wrapped, kek)
```

## Android 的 FDE 历程

FDE加密又是怎么一回事呢，看看google怎么定义的吧（以下部分内容引用网络某大神的解释）：

- 安卓会通过一个随机生成的128位设备加密密钥 (Device Encryption Key, DEK) 来加密设备的文件系统。
- 安卓使用用户的PIN或者密码来加密DEK，并将它存储在设备加密过的文件系统上。从物理上来讲，它也在设备的闪存芯片中。当你输入正确的PIN或密码时，设备可以解锁DEK，并使用密钥来解锁文件系统。

- 实际上，DEK是使用用户的PIN或密码，外加一个被称为KeyMaster Key Blob的加密数据块来进行加密的。这个数据块包含一个由KeyMaster程序生成的2048位RSA密钥，它运行在设备处理器上的一个安全区域上。KeyMaster会创建RSA密钥，将其存储在数据块中，并为安卓系统创建一份加密过的拷贝版本。
    安卓系统和你的移动应用运行在处理器的非安全区域上。安卓没有访问KeyMaster的安全世界的权限，因此它无法知晓数据块里的RSA密钥。安卓只能获得这个数据块的加密版本，而只有KeyMaster能够解密它。

- 当你输入PIN或密码时，安卓拿到加密过的数据块，并将它和使用scrypt处理过的PIN或密码一起，传回运行在处理器安全区域上的KeyMaster。KeyMaster将私密地使用处理器中带有的私钥来对数据块进行解密，获得长RSA密钥。然后，它将私密地使用scrypt处理过的PIN或密码，外加长RSA密钥，来制造一个RSA签名，并将签名发回给安卓。之后安卓使用一系列算法来处理这一签名，并最终解密DEK，解锁设备。

因此，全部流程都基于KeyMaster的数据块。数据块包含解密DEK所需的长RSA密钥。安卓只拥有加密后的数据块，而只有用户才有PIN或密码。此外，只有KeyMaster才能解密加密过的数据块。

### Google 的版本历史

- Android 4：首次引入 FDE 的 beta版本，由于 CPU 普遍不支持硬件加密严重影响速度，并未普遍应用
- Android 5: Google 推出 5.0 强制采用 FDE 加密，由于终端厂商的普遍抵制又推出 5.1 版本，改为用户自行设置是否启用
- Android 6: 高通发布骁龙810支持硬件加密，6.0 版本终于强制采用 FDE 加密

### 5.0中的加密 磁盘加密密钥 的实现逻辑

1. 产生随机16 Bytes DEK(disk encryption key--磁盘加密用的密钥)及16 Bytes SALT；
2. 对(用户密码+SALT)使用scrypt算法产生32 Bytes HASH 作为IK1(intermediate key 1);
3. 将IK1填充到硬件产生的私钥规格大小(目前看到是RSA算法，256Bytes), 具体是: 
00 || IK1 || 00..00  ## one zero byte, 32 IK1 bytes, 223 zero bytes.
4. 使用硬件私钥 HBK 对 IK1 进行签名，生成256 Bytes签名数据作为IK2；
5. 对(IK2+SALT)使用scrypt算法(与第二步中的SALT相同)产生出32 Bytes HASH 作为IK3；
6. 使用IK3前16 Bytes作为KEK(用来加密主密钥DEK的KEY)，后16 Bytes作为算法IV(初始化向量)；
7. 使用AES_CBC算法，采用KEK作为密钥，IV作为初始化向量来加密用户的主密钥DEK，生成加密后的主密钥，存入分区尾部数据结构中；

---

Secure Startup
Secure Startup 為 FDE 的延伸版本，唯一的差別是保護加密/解密的 Masterkey 金鑰的方式不同。原本在 FDE 架構下保護 Masterkey 的其中一個密碼為 default_password，在 Secure Startup 下可改為用戶自訂的螢幕鎖密碼。


## AFPS

![AFPS](apfs_concepts.png)

[https://bombich.com/kb/ccc5/working-apfs-volume-groups](https://bombich.com/kb/ccc5/working-apfs-volume-groups)
[https://www.ntfs.com/apfs-structure.htm](https://www.ntfs.com/apfs-structure.htm)
---

## 参考文献

Stephen Foskett 是一名存储技术专家，其个人博客对 FileVault 进行了非常有价值的分析。
请参见[MacOS X 的 Corestorage 逻辑卷组管理器](https://blog.fosketts.net/2011/08/04/mac-osx-lion-corestorage-volume-manager/)

- [MacOS 的安全和隐私指南](https://github.com/drduh/macOS-Security-and-Privacy-Guide/blob/master/README-cn.md)
- [FileVault2加密分区离线解密技术及其取证应用 - 蓝朝祥](FileVault2加密分区离线解密技术及其取证应用_蓝朝祥.pdf)
- [Security Analysis and Decryption of FileVault 2 - Omar Choudary](slides_fv2_ifip_2013.pdf)
- [Infiltrate the Vault: Security Analysis and Decryption of Lion Full Disk Encryption - Omar Choudary](2012-374.pdf)
- [你的安卓手机究竟是FDE加密还是FBE加密？](https://page.om.qq.com/page/O3yauEIx2l-9WrUkHgQgRUBw0)
- [android FDE功能介绍](https://blog.csdn.net/bob_fly1984/article/details/80369900)
