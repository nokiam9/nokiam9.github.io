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

1. 启动卷宗的分区改为一个 CoreStorage 管理的加密卷，并增加了一个 Recovery HD 分区卷；
2. 加密标准改为 NIST 推荐的 AES-XTS，分组长度为128位，密钥长度为256位；
3. 支持**用户登录密码**作为加密因子；

![LVF](lvf.jpg)

CoreStorage 是 Apple 开发的 LVM（Logic Volume Manager，逻辑卷管理器），作为磁盘和文件系统之间的虚拟化层，增加了操作系统存储分配的灵活性，类似于赛门铁克的 Veritas Volume Manager 和 OSF LVM，但底层文件系统仍然是 HFS+。
Apple 还引入了一个新概念 LVF（logical volume family，逻辑卷系列），用于指定将由它所包含的逻辑卷继承的属性（目前只有 FileVault），但以后可能用于新的性能特征。
现在，当使用 FileVault 2 加密时，MacOS 会自动将已有数据卷转换为 CoreStorage 卷，并将分区封装为 PV，将其导入 LVG，并设置 LVF 和 LV 以包含新的文件系统。
对于支持 AES 指令集的 CPU（如Intel Broadwell 架构），AES-128-XTS 加密模式只有 3% 左右的性能损耗，但对于不支持该指令集的 CPU（如早期的酷睿CPU）会有明显的性能下降。

### AES-XTS的密钥层次架构

FileValut 2 设计了多个不同类型的密钥。
![密钥层次架构](arch.png)

#### 1. Volume Master Key（VMK）- 分组密钥 key1

密钥长度：128位
构造方式：**随机生成**
存储位置：Recovery HD -> `EncryptedRoot.plist` -> `KEKWrappedVolumeKeyStruct`字段加密存储

#### 2. Volume tweak key - 可调整密钥 key2

密钥长度：128位
构造方式：$trunc_{128}(SHA256(VolumeMasterKey || LogicVolumeFamilyIdentifier))$

#### 3. EncryptedRoot.plist.wipekey - 密钥文件

在逻辑卷 Recovery HD 中，有一个加密文件包含了提取 VMK 所需的全部信息，路径是`com.apple.boot.X/System/Library/Caches/com.apple.corestorage/EncryptedRoot.plist.wipekey`

这个文件本身也是基于 AES-XTS 加密的，可调整密钥设置为 128位的全零，分组密钥保存在 CoreStorage header 的`Physical Volume Identifier`字段中，请参见[Infiltrate the Vault: Security Analysis and Decryption of Lion Full Disk Encryption - Omar Choudary](2012-374.pdf)论文附录（第13页）的 Table 2。

plist文件解密成功后，其内容示例见下：

```xml
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
        <key>ConversionInfo</key>
        <dict>
                <key>ConversionStatus</key>
                <string>Complete</string>
                <key>TargetContext</key>
                <integer>1</integer>
        </dict>
        <key>CryptoUsers</key>
        <array>
                <dict>
                        ...
                        <key>PassphraseWrappedKEKStruct</key>
                        <data>
                        ...
                        </data>
                        ...
                </dict>
                <dict>
                ...
                </dict>
        </array>
        <key>LastUpdateTime</key>
        <integer>1323243315</integer>
        <key>WrappedVolumeKeys</key>
        <array>
                <dict>
                        <key>BlockAlgorithm</key>
                        <string>None</string>
                        <key>KEKWrappedVolumeKeyStruct</key>
                        <data>
                        </data>
                        ...
                </dict>
                <dict>
                        <key>BlockAlgorithm</key>
                        <string>AES-XTS</string>
                        <key>KEKWrappedVolumeKeyStruct</key>
                        <data>
                        ...
                        </data>
                        ...
                </dict>
        </array>
</dict>
</plist>
```

其中的关键字段如下，注意实际存储格式是 base64 编码：

- PassphraseWrappedKEKStruct(1)：284 位的 recovery password
- PassphraseWrappedKEKStruct(2)：284 位的 user password
- KEKWrappedVolumeKeyStruct(1)：未使用！标记为 None
- KEKWrappedVolumeKeyStruct(2)：VMK 的包裹信息，标记为 AES-XTS

上述 struct 的结构定义，请参见[Infiltrate the Vault: Security Analysis and Decryption of Lion Full Disk Encryption - Omar Choudary](2012-374.pdf)论文附录（第13页）的 Table 3 和 Table 4

#### 4. Recovery key - 恢复密钥

启用 FileVault 时，系统自动生成并要求用户保存`Recovery key`，此密钥可以用来解封 VMK。
![RE](recovery-key.png)
使用 PBKDF2 算法恢复 VMK（注意恢复密钥是字符串格式，包括数字之间的破折号）：

``` c
PK = PBKDF2( PRF=SHA256, password=Recovery key,
    salt=PassphraseWrappedKEKStruct.salt,
    iters=41000, dk_len=128)
KEK = AES_Unwrap(PassphraseWrappedKEKStruc.AES-wrapped-volume-KEK, PK)
VMK = AES_Unwrap(KEKWrappedVolumeKeyStruct.AES-wrapped-volume-key, KEK)
```

迭代次数存储在`PassphraseWrappedKEKStruct`中，但对于 Mac OS 10.7 似乎始终为 41000。

#### 5. User Key - 用户密码

对于 Mac OS 系统上的每个用户，FileVault 使用各自的用户密码（目前仅包含 ASCII 字符）来计算用户密钥，并解锁加密数据。
每个用户都有其自己关联的 PassphraseWrappedKEKStruct，按照与恢复密钥相同的方式计算并使用相应的用户密钥来获取卷主密钥。

#### 6. 小结

上述过程以伪代码表示如下：

```c
p = get_user_password()
salt = get_salt_from_PassphraseWrappedKEK()
iterations = 41000
pk = pbkdf2(p, salt, iterations, HMAC-SHA256)
kek_wrapped = get_kek_from_PassphraseWrappedKEK()
kek = aes_unwrap(kek_wrapped, pk)
vmk_wrapped = get_vmk_from_KEKWrappedVolumeKey()
vmk = aes_unwrap(vmk_wrapped, kek)
```

## 三、基于 APFS 的 FileVault

2017年，MacOS High Sierra（内华达脊岭，10.13） 发布了 APFS（Apple FileSystem），用于替代古老的 HFS+，并完全整合了 CoreStorage，实现了原生的 FileVault。

APFS 引入了 container（容器）的概念，使用 GPT 分区方案，单一容器内部包含多个 Volume，这些逻辑卷共享物理存储容量，并可以相互读取数据，但是不能与其他容器共享数据。

- GPT 方案中有一个或多个 APFS 容器
- 每个容器内都有一个或多个 APFS 卷，所有这些卷共享分配给容器的空间，并且每个卷都可以具有 APFS 卷的角色
- MacOS 10.15 Catalina 引入了 APFS Volume Group，在 Finder 中显示为单个卷
- MacOS 10.15 Catalina 中，系统卷角色（通常名为“Macintosh HD”）设为只读，而在 MacOS 11 Big Sur 中，它变为签名系统卷 (SSV)，并且仅挂载卷快照。
- 数据卷角色（通常称为 Macintosh HD - 数据）用作系统卷的覆盖或影子，其中系统卷和数据卷属于同一卷组，并且在 Finder 中被视为一个卷

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

- Volume disk1s5(System)：系统主启动卷，挂载点位于`/`，FileVault = Yes (Unlocked)；后续版本还有 snapshot 快照功能
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

进一步，可以通过`diskutil apfs listusers $Volume_ID`，查看当前有效的用户密钥信息。

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

## 参考文献

- [Legacy FileVault 漏洞报告](https://fahrplan.events.ccc.de/congress/2006/Fahrplan/attachments/1244-23C3VileFault.pdf)
- [MacOS X 的 Corestorage 逻辑卷组管理器 - Stephen Foskett](https://blog.fosketts.net/2011/08/04/mac-osx-lion-corestorage-volume-manager/)
- [MacOS 的安全和隐私指南](https://github.com/drduh/macOS-Security-and-Privacy-Guide/blob/master/README-cn.md)
- [Working with APFS Volume Groups](https://bombich.com/kb/ccc5/working-apfs-volume-groups)
- [APFS 科普贴](https://www.jianshu.com/p/c401d546cebf)
- [APFS Structure](https://www.ntfs.com/apfs-structure.htm)
- [FileVault Drive Encryption 代码库 - Github](https://github.com/libyal/libfvde/blob/main/documentation/FileVault%20Drive%20Encryption%20(FVDE).asciidoc)

### 文档下载

- [FileVault2加密分区离线解密技术及其取证应用 - 蓝朝祥](FileVault2加密分区离线解密技术及其取证应用_蓝朝祥.pdf)
- [Security Analysis and Decryption of FileVault 2 - Omar Choudary](slides_fv2_ifip_2013.pdf)
- [Infiltrate the Vault: Security Analysis and Decryption of Lion Full Disk Encryption - Omar Choudary](2012-374.pdf)
