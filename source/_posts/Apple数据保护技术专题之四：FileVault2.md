---
title: Apple数据保护技术专题之四：FileVault2
date: 2022-12-08 11:19:15
tags:
---

从本质上说，FileVault 属于 FDE（Full Disk Encryption，全盘加密）技术，早期的 Android 设备也是采用此方案，但随着技术发展，Andriod 和 iOS 逐渐都演进为 FBE（File Based Encryption，文件加密），仅在 MacOS 中保留了 FileVault2，但同时也支持 DataProtection 数据保护技术。

## 一、Legacy FileVault

FileVault 首次随附于 Mac OS X Panther (10.3),当时的版本只允许加密用户目录，而非启动宗卷。操作系统使用一个加密的稀疏磁盘镜像（一个单一的大文件）为用户目录的数据提供虚拟磁盘。 Mac OS X Leopard 和 Mac OS X Snow Leopard 使用更现代的 稀疏磁盘镜像包，将数据拆分成8 MB大小的文件（被称作 bands）存放于文件包中。 Apple将采用这两种方法加密的版本称为legacy FileVault。

原始版本的 FileVault 随附于Mac OS X Panther，且只被用于加密用户目录。
启动 FileVault 功能时，系统会要求用户设置一个**主密码**，当用户忘记密码时，便可以使用这个主密码或是恢复密钥直接解密文件。

legacy FileVault已被证明存在一些缺陷，通过破解1024-bit RSA 或 3DES-EDE算法，加密数据可被解密。

和 Legacy FileVault 使用的 CBC 操作模式比较， (请参阅磁盘加密原理)， FileVault 2 使用的 XTS-AESW 模式更健壮。 其他问题包括将macOS置于睡眠模式时，密钥存在的泄露风险。

### 二、FileVault 2

Mac OS X Lion (10.7) 及之后的系统随附重新设计过的 FileVault 2，新版的软件将直接加密整个OS X的启动宗卷（自然也包括用户目录），并抛弃了之前使用磁盘镜像的方法。采用这种加密方法，已授权的用户信息将从一个单独的非加密引导卷宗读取。(分区格式为 Apple_Boot)。

FileVault使用**用户登录密码**作为加密令牌，并采用AES的XTS-AES模式将数据划分成128位的块，同时生成一枚256位密钥来加密磁盘，这一标准也是NIST推荐的标准。当用户解锁后，其他用户也可以访问这些数据，直到电脑关机。

约有3%的I/O的性能下降会出现在使用AES指令集的CPU上（如Intel Broadwell 架构CPU， OS X 10.10.3环境），早期的酷睿CPU，或者其他不使用该指令集的处理器会有更明显的性能下降。

系统运行时执行 FileVault2，电脑会创建一个**恢复密钥**，并在屏幕上显示出来，提示用户保管，并且提供了一个将密钥上传至 Apple 的可选项。恢复密钥共有120位，由全部英文字母和数字 1-9 组成，系统会调用/dev/random的随机数生成器生成整枚密钥。因此其安全性取决于macOS 使用的随机数生成算法，根据密码学家 2012 年的声明指出，该算法是安全的。

除非重新加密整个卷宗，否则将无法修改恢复密钥。

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

---

## 参考文献

- [MacOS 的安全和隐私指南](https://github.com/drduh/macOS-Security-and-Privacy-Guide/blob/master/README-cn.md)
- [FileVault2加密分区离线解密技术及其取证应用 - 蓝朝祥](FileVault2加密分区离线解密技术及其取证应用_蓝朝祥.pdf)
- [Security Analysis and Decryption of FileVault 2 - Omar Choudary](slides_fv2_ifip_2013.pdf)
- [Infiltrate the Vault: Security Analysis and Decryption of Lion Full Disk Encryption - Omar Choudary](2012-374.pdf)
- [你的安卓手机究竟是FDE加密还是FBE加密？](https://page.om.qq.com/page/O3yauEIx2l-9WrUkHgQgRUBw0)
- [android FDE功能介绍](https://blog.csdn.net/bob_fly1984/article/details/80369900)
