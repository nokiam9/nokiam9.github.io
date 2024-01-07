---
title: Apple FileVault的技术分析
date: 2024-01-07 16:07:04
tags:
---

FileVault 的意思是文件保管箱，是 Apple 公司为 Mac 电脑提供的、基于 MacOS 的加密技术方案，其本质归类于 FDE（Full Disk Encryption，全盘加密）。
注意！与之对应的，为 iPhone 提供的、基于 iOS 的 DataProtect （数据保护）加密技术方案，其本质归类于 FBE（File Based Encryption，文件加密）。

## 一、Legacy FileVault

2003年，Mac OS X Panther (黑豹，10.3) 首次发布了 FileVault，当时的版本只加密了用户的主目录（不包含启动宗卷），方法是采用一个巨大的稀疏磁盘镜像文件（sparse disk image，区别于普通的磁盘镜像文件`.dmg`，其实际磁盘占用空间是根据使用情况动态扩展的）作为虚拟磁盘，负责存储加密后的文件数据。在启用 FileVault 功能时，系统会要求用户设置一个**主密码**，当用户忘记密码时，需要这个主密码或是恢复密钥直接解密文件。

2006年，Mac OS X Leopard（花豹，10.5）做了少许改进，将虚拟磁盘改为稀疏绑定磁盘镜像（sparse bundle disk image，其特点是将文件分割为若干个 8MB 的 Band，在文件修改时无需复制全部内容，仅复制本次修改的 Band，因此备份速度更快，非常适合 Time Machine 的需求）。

Apple 将上述两种版本的技术统称为 **Legacy FileVault** ，其存在很多严重问题，包括：

- 仅加密用户目录`$home`是远远不够的，`/tmp`和`/var/log`等目录怎么办？
- 主密钥存储在磁盘镜像文件的头部，密钥包裹算法 3DES-EDE 存在破解风险；恢复密钥的包裹算法是 1024-bit RSA，加密强度不高，也存在破解风险
- 文件内容加密采用 AES-128 + CBC 工作模式，但初始向量 IV 没有随机性，设置算法被破解：$IV=trunc_{128}(HMACSHA1(hmac-key||trunkno))$
- MacOS 置于睡眠模式时，内存写入`/var/vm/sleepimage`，存在密钥泄露风险

这些问题的根源，还是由于底层文件系统 HFS+ 不支持 FDE 加密，只能基于“应用软件模拟 + 专用磁盘镜像”的替代方式实现，导致功能受限而且效率低下。

## 二、基于 CoreStorage 的 FileVault 2

2010年，Mac OS X Lion (狮子，10.7) 发布了 FileVault 2，推出一个重新设计的 FDE 方案。

![密钥层次架构](arch.png)

1. 启动卷宗的分区改为一个 CoreStorage 管理的加密卷，终于可以全盘加密了！
2. 增加了一个 Recovery HD 分区卷，但仍然是 HFS+ 文件系统；
3. 加密标准改为 NIST 推荐的 AES-XTS，分组长度为128位，密钥长度为256位；
4. 支持**用户登录密码**作为加密因子；

CoreStorage 是 Apple 开发的 LVM（Logic Volume Manager，逻辑卷管理器），类似于赛门铁克的 Veritas Volume Manager 和 OSF LVM。

LVM 作为磁盘和文件系统之间的虚拟化层，增加了操作系统存储分配的灵活性，因为现代计算机系统需要保持一致的文件系统映像，即使存储设备发生变化也是如此。
Apple 引入了一个新概念 LVF（logical volume family，逻辑卷系列），用于指定将由它所包含的逻辑卷继承的属性（目前只有 FileVault ），但以后可能用于新的性能特征。
现在，当使用 FileVault 2 加密时，MacOS 会自动将已有数据卷转换为 CoreStorage 卷，并将分区封装为 PV，将其导入 LVG，并设置 LVF 和 LV 以包含新的文件系统。

![LVF](lvf.jpg)

对于支持 AES 指令集的 CPU（如Intel Broadwell 架构），AES-128-XTS 加密模式只有 3% 左右的性能损耗，但对于不支持该指令集的 CPU（如早期的酷睿CPU）会有明显的性能下降。

## 三、基于 APFS 的 FileVault

2017年，MacOS High Sierra（内华达脊岭，10.13） 发布了 APFS（Apple FileSystem），用于替代古老的 HFS+，并整合了基于 CoreStorage 的 FileValut 2。
APFS 引入了 container（容器）的概念，单一容器内部包含多个 Volume，这些逻辑卷共享物理存储容量，并可以相互读取数据，但是不能与其他容器共享数据。

![a](apfs.png)

命令行`df -h`查看文件系统

```console
sj@JiandeiMac ~ % df -h
Filesystem      Size   Used  Avail Capacity iused      ifree %iused  Mounted on
/dev/disk1s5   234Gi   10Gi   29Gi    27%  488433 2448636927    0%   /
devfs          190Ki  190Ki    0Bi   100%     659          0  100%   /dev
/dev/disk1s1   234Gi  191Gi   29Gi    87% 3218923 2445906437    0%   /System/Volumes/Data
/dev/disk1s4   234Gi  2.0Gi   29Gi     7%       2 2449125358    0%   /private/var/vm
map auto_home    0Bi    0Bi    0Bi   100%       0          0  100%   /System/Volumes/Data/home
/dev/disk1s3   234Gi  505Mi   29Gi     2%      50 2449125310    0%   /Volumes/Recovery
```

通过`diskutil apfs list`，可以查看容器 disk1 及 物理设备 disk0s2 的详细信息，依次包括：

- Volume disk1s5(System)：挂载点位于`/`，FileVault = Yes (Unlocked)
- volume disk1s1(**Data**)：加密的用户数据，挂载点位于`/System/Volumes/Data`，FileVault = Yes (Unlocked)
- Volume disk1s4(VM)：保存休眠状态，挂载点位于`/private/var/vm`，FileVault = No
- Volume disk1s3(Revovery)：进入macOS的recovery模式，可以用来清除用户密码，挂载点位于`/Volumes/Recovery`，FileVault = No
- Volume disk1s2(PreBoot)：如果启用了FileVault加密会通过这个分区启动系统，**当前未挂载**，FileVault = No

```console
sj@JiandeiMac ~ % diskutil apfs list
APFS Container (1 found)
|
+-- Container disk1 0B311DB5-B224-4F51-AA7C-1211E5A2A994
    ====================================================
    APFS Container Reference:     disk1
    Size (Capacity Ceiling):      250790436864 B (250.8 GB)
    Capacity In Use By Volumes:   219201548288 B (219.2 GB) (87.4% used)
    Capacity Not Allocated:       31588888576 B (31.6 GB) (12.6% free)
    |
    +-< Physical Store disk0s2 4024090C-7938-4238-9772-192071FEDE07
    |   -----------------------------------------------------------
    |   APFS Physical Store Disk:   disk0s2
    |   Size:                       250790436864 B (250.8 GB)
    |
    +-> Volume disk1s1 925F8706-12D2-305B-B8E0-14201AF1D027
    |   ---------------------------------------------------
    |   APFS Volume Disk (Role):   disk1s1 (Data)
    |   Name:                      未命名 - 数据 (Case-insensitive)
    |   Mount Point:               /System/Volumes/Data
    |   Capacity Consumed:         205089447936 B (205.1 GB)
    |   FileVault:                 Yes (Unlocked)
    |
    +-> Volume disk1s2 35818BF3-204E-4024-A049-FE5D61D96B74
    |   ---------------------------------------------------
    |   APFS Volume Disk (Role):   disk1s2 (Preboot)
    |   Name:                      Preboot (Case-insensitive)
    |   Mount Point:               Not Mounted
    |   Capacity Consumed:         81448960 B (81.4 MB)
    |   FileVault:                 No
    |
    +-> Volume disk1s3 3873B186-B246-4F1D-8E7B-4E2515E4B838
    |   ---------------------------------------------------
    |   APFS Volume Disk (Role):   disk1s3 (Recovery)
    |   Name:                      Recovery (Case-insensitive)
    |   Mount Point:               /Volumes/Recovery
    |   Capacity Consumed:         529969152 B (530.0 MB)
    |   FileVault:                 No
    |
    +-> Volume disk1s4 F1A4B4C8-0360-4F92-8687-9BCE3F2F9134
    |   ---------------------------------------------------
    |   APFS Volume Disk (Role):   disk1s4 (VM)
    |   Name:                      VM (Case-insensitive)
    |   Mount Point:               /private/var/vm
    |   Capacity Consumed:         2148556800 B (2.1 GB)
    |   FileVault:                 No
    |
    +-> Volume disk1s5 7B89960E-7BBB-4012-BD70-27E4DE0A5ADC
        ---------------------------------------------------
        APFS Volume Disk (Role):   disk1s5 (System)
        Name:                      未命名 (Case-insensitive)
        Mount Point:               /
        Capacity Consumed:         11213705216 B (11.2 GB)
        FileVault:                 Yes (Unlocked)
```

> 如果 FileVault 未启用，FileVault = No (Encrypted at rest)

进一步，可以通过`diskutil apfs listusers $ID`，查看当前有效的用户密钥信息。

```console
sj@JiandeiMac /Volumes % diskutil apfs listusers 925F8706-12D2-305B-B8E0-14201AF1D027
Cryptographic users for disk1s1 (3 found)
|
+-- FBD4D606-E5F2-4FC5-B6C0-70E11D1A3FB1
|   Type: Local Open Directory User
|
+-- EC1C2AD9-B618-4ED6-BD8D-50F361C27507
|   Type: iCloud Recovery User
|
+-- 64C0C6EB-0000-11AA-AA11-00306543ECAC
    Type: iCloud Recovery External Key
```


---

## 附录一：离线解密方法

系统运行时执行 FileVault2，电脑会创建一个**恢复密钥**，并在屏幕上显示出来，提示用户保管，并且提供了一个将密钥上传至 Apple 的可选项。
恢复密钥共有120位，由全部英文字母和数字 1-9 组成，系统会调用`/dev/random`的随机数生成器生成整枚密钥。

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

## 参考文献

- [Legacy FileVault 漏洞报告](https://fahrplan.events.ccc.de/congress/2006/Fahrplan/attachments/1244-23C3VileFault.pdf)
- [MacOS X 的 Corestorage 逻辑卷组管理器 - Stephen Foskett](https://blog.fosketts.net/2011/08/04/mac-osx-lion-corestorage-volume-manager/)
- [MacOS 的安全和隐私指南](https://github.com/drduh/macOS-Security-and-Privacy-Guide/blob/master/README-cn.md)
- [Working with APFS Volume Groups](https://bombich.com/kb/ccc5/working-apfs-volume-groups)
- [APFS 科普贴](https://www.jianshu.com/p/c401d546cebf)
- [APFS Structure](https://www.ntfs.com/apfs-structure.htm)

### 文档下载

- [FileVault2加密分区离线解密技术及其取证应用 - 蓝朝祥](FileVault2加密分区离线解密技术及其取证应用_蓝朝祥.pdf)
- [Security Analysis and Decryption of FileVault 2 - Omar Choudary](slides_fv2_ifip_2013.pdf)
- [Infiltrate the Vault: Security Analysis and Decryption of Lion Full Disk Encryption - Omar Choudary](2012-374.pdf)

---

- [你的安卓手机究竟是FDE加密还是FBE加密？](https://page.om.qq.com/page/O3yauEIx2l-9WrUkHgQgRUBw0)
- [android FDE功能介绍](https://blog.csdn.net/bob_fly1984/article/details/80369900)
