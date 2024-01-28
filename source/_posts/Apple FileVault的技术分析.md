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
- MacOS 置于睡眠模式时，内存数据直接写入磁盘文件`/var/vm/sleepimage`，缺乏加密保护存在密钥泄露风险

这些问题的根源，还是由于底层文件系统 HFS+ 不支持 FDE 加密，只能基于“应用软件模拟 + 专用磁盘镜像”的替代方式实现，导致功能受限而且效率低下。

## 二、基于 CoreStorage 的 FileVault 2

2010年，Mac OS X Lion (狮子，10.7) 发布了 FileVault 2，基于 CoreStorage 虚拟化技术推出一个重新设计的 FDE 方案，主要改造点包括：

- 随着 NIST 技术标准的演进，将 AES-CBC 替换为 AES-XTS 磁盘加密模式，分组长度为128位，密钥长度为256位，并支持基于 AES 指令集的硬件解密（如Intel Broadwell 架构的CPU，加密模式只有3%的性能损耗）。
- 在磁盘和文件系统之间增加了一个虚拟化层，也就是命名为 CoreStorage 的 LVM （Logic Volume Manager，逻辑卷管理器），类似于 Veritas Volume Manager 和 OSF LVM，但底层文件系统仍然是 HFS+。
- 支持将**User Password**作为加密因子，而且支持 MacOS 的每个用户使用各自的用户密码来计算用户密钥，并解锁加密数据，非常有利于用户隐私保护！

### 1. 启用方式

当用户启用 FileVault 时，主要任务包括：

1. 系统自动生成并要求用户保存`Recovery password`
    ![RE](recovery-key.png)
2. 自动将已有数据卷转换为 CoreStorage 加密卷，并将分区封装为 PV，将其导入 LVG，并设置 LVF 和 LV 以包含新的文件系统。
3. 新建一个 Recovery HD 分区卷，并将 CoreStorage 加密卷设置为启动分区。

```console
Mikes-MacBook-Pro-3:~ mikej$ diskutil list
/dev/disk0 (internal, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *251.0 GB   disk0
   1:                        EFI EFI                     209.7 MB   disk0s1
   2:          Apple_CoreStorage Mike HD                 250.1 GB   disk0s2
   3:                 Apple_Boot Recovery HD             650.1 MB   disk0s3
```

### 2. 技术实现

通过`diskutil corestorage list`命令，可以查看 CoreStorage 的层级结构为:
**Physical Volume -> Logical Volume Group -> Logical Volume Family -> Logical Volume**

```console
CoreStorage logical volume groups (1 found)
|
+-- Logical Volume Group 5D6504C7-6C94-498E-B50C-64E3E4950AE0
|   =========================================================
|   Name:         Macintosh HD
|   Status:       Online
|   Size:         125318770688 B (125.3 GB)
|   Free Space:   0 B (0 B)
|   |
|   +-< Physical Volume 499AA4FC-31C1-47A3-8364-89A0C836125C        # 上层PV
|   |   ----------------------------------------------------
|   |   Index:    0
|   |   Disk:     disk0s2
|   |   Status:   Online
|   |   Size:     125318770688 B (125.3 GB)
|   |
|   +-> Logical Volume Family 163A0B82-4315-4C68-8403-52B5A918C57C  # 下层LVF
|       ----------------------------------------------------------
|       Encryption Status:       Unlocked
|       Encryption Type:         AES-XTS
|       Conversion Status:       Complete
|       Conversion Direction:    -none-
|       Has Encrypted Extents:   Yes
|       Fully Secure:            Yes
|       Passphrase Required:     Yes
|       |
|       +-> Logical Volume 264CFBDC-8103-47E0-978E-738789320980     # 再下层LV
|           ---------------------------------------------------
|           Disk:                  disk1
|           Status:                Online
|           Size (Total):          124999999488 B (125.0 GB)
|           Conversion Progress:   -none-
|           Revertible:            Yes (unlock and decryption required)
|           LV Name:               Macintosh HD
|           Volume Name:           Macintosh HD
|           Content Hint:          Apple_HFS
```

LVF- logical volume family 是 Apple 自定义的层级，用于 LV 逻辑卷特定属性的继承。
系统变量`com.apple.corestorage.lv.familyUUID`用于构造 AES-XTS 的 Tweak key 的加密因子。

## 三、基于 APFS 的 FileVault

2017年，MacOS High Sierra（内华达脊岭，10.13） 正式启用了 APFS（Apple FileSystem），用于替代古老的 HFS+。
作为新一代的文件系统，APFS 专门针对闪存/SSD进行优化（但依然可用于传统机械硬盘），提供了更强大的加密、写入时复制元数据、空间分享、文件和目录克隆、快照、目录大小快速调整、原子级安全存储基元，以及改进的文件系统底层技术，将全面应用于该公司旗下 MacOS、iOS、iPadOS、tvOS、watchOS等所有设备中。其突出特点包括：

- inode编码长度提高到64位，单一Volume的文件数量大大增加；时间戳精度提高到纳秒，有助于实现原子性和原子事务；目录大小是单独存储的，无需每次实时计算目录容量；文件和文件夹名称被规范化，完全支持Unicode
- 支持COW（Copy On Write，写入时复制）：几乎立即复制文件或目录，元数据多次存在于文件结构中，但共享相同的数据存储空间；修改克隆时，文件系统仅记录数据更改
- 支持快照（Snapshot）：支持创建特点时间点、文件系统只读实例的快照
- 支持空间共享：使用 GPT 分区方案，单一容器内部包含多个 Volume 共享物理存储容量，并可以相互访问，但不与其他容器共享数据

![a](apfs.png)

关于加密功能，APFS 引入 container（容器）并整合了 CoreStorage 技术，完全实现了文件系统层级的原生实现，包括：

1. 支持对容器、卷和文件使用的数据结构进行加密。当一个volume是加密的，它的文件系统树和该卷中的文件内容都是加密的
2. 统一实现了 FDE 和 FBE，支持三种加密模型：不加密、单密钥加密、多密钥加密，但**MacOS 目前似乎仅支持 FDE 模式**
3. 支持硬件加密和软件加密。
    硬件加密适用于Apple提供的内置存储（例如带有T2安全芯片的 MacOS 和 iOS 设备）；
    软件加密适用于用于外部存储，以及不支持硬件加密的设备上的内部存储；
    使用硬件加密时，只有操作系统内核可以与内部存储交互；
    根据硬件的不同，可以使用 AES-XTS 或 AES-CBC 加密模式。

层级结构也调整为：**Physical Store -> Contianer -> （Volume Group） -> Volume**

### 1. 技术分析（Intel CPU）

示例是一台 Intel CPU 的 iMac 设备，因此并没有安全隔区等 Apple 专用 Soc 设备。
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

通过`diskutil list`，可以看到物理硬盘`disk0`包含了2个分区，分别是EFI启动分区`disk0-s1`和APFS分区`disk0-s2`，并将后者同步映射到`disk1`。

```console
j@JiandeiMac ~ % diskutil list
/dev/disk0 (internal, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *251.0 GB   disk0
   1:                        EFI EFI                     209.7 MB   disk0s1
   2:                 Apple_APFS Container disk1         250.8 GB   disk0s2

/dev/disk1 (synthesized):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      APFS Container Scheme -                      +250.8 GB   disk1
                                 Physical Store disk0s2
   1:                APFS Volume 未命名 - 数据           205.4 GB   disk1s1
   2:                APFS Volume Preboot                 81.4 MB    disk1s2
   3:                APFS Volume Recovery                530.0 MB   disk1s3
   4:                APFS Volume VM                      5.4 GB     disk1s4
   5:                APFS Volume 未命名                  11.2 GB    disk1s5
```

![AFPS](apfs_concepts.png)
用于启动 Mac 的 APFS 容器必须包含至少五个宗卷，其中前三个宗卷对用户隐藏 ：

- Preboot Volume：包含启动容器中每个系统宗卷所需的数据
- VM Volume：MacOS 用于交换文件储存
- Recovery Volume：包含 recoveryOS，进入Recovery模式可以清除用户密码
- System Volume：包含用于启动 Mac 的所有必要文件、macOS 原生安装的所有 App
- Data Volume：包含用户文件夹中的任何数据、用户安装的 App、第三方 App、用户拥有且能够写入的其他位置

每增加一个系统宗卷， 便会创建一个数据宗卷。3个隐藏宗卷全为共享宗卷且无法复制。
通过`diskutil apfs list`，可以查看容器 disk1 的详细信息。

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
    +-< Physical Store disk0s2 4024090C-7938-4238-9772-192071FEDE07     # 上层物理设备
    |   -----------------------------------------------------------
    |   APFS Physical Store Disk:   disk0s2
    |   Size:                       250790436864 B (250.8 GB)
    |
    +-> Volume disk1s1 925F8706-12D2-305B-B8E0-14201AF1D027             # 数据Volume
    |   ---------------------------------------------------
    |   APFS Volume Disk (Role):   disk1s1 (Data)
    |   Name:                      未命名 - 数据 (Case-insensitive)
    |   Mount Point:               /System/Volumes/Data                 # 加载用户数据目录
    |   Capacity Consumed:         205089447936 B (205.1 GB)
    |   FileVault:                 Yes (Unlocked)                       # 加密状态：已解锁
    |
    +-> Volume disk1s2 35818BF3-204E-4024-A049-FE5D61D96B74             # 预启动Volume
    |   ---------------------------------------------------
    |   APFS Volume Disk (Role):   disk1s2 (Preboot)
    |   Name:                      Preboot (Case-insensitive)
    |   Mount Point:               Not Mounted
    |   Capacity Consumed:         81448960 B (81.4 MB)
    |   FileVault:                 No
    |
    +-> Volume disk1s3 3873B186-B246-4F1D-8E7B-4E2515E4B838
    |   ---------------------------------------------------
    |   APFS Volume Disk (Role):   disk1s3 (Recovery)                   # 恢复Volume
    |   Name:                      Recovery (Case-insensitive)
    |   Mount Point:               /Volumes/Recovery
    |   Capacity Consumed:         529969152 B (530.0 MB)
    |   FileVault:                 No
    |
    +-> Volume disk1s4 F1A4B4C8-0360-4F92-8687-9BCE3F2F9134             # 虚拟内存Volume
    |   ---------------------------------------------------
    |   APFS Volume Disk (Role):   disk1s4 (VM)
    |   Name:                      VM (Case-insensitive)
    |   Mount Point:               /private/var/vm
    |   Capacity Consumed:         2148556800 B (2.1 GB)
    |   FileVault:                 No
    |
    +-> Volume disk1s5 7B89960E-7BBB-4012-BD70-27E4DE0A5ADC             # 系统Volume
        ---------------------------------------------------
        APFS Volume Disk (Role):   disk1s5 (System)
        Name:                      未命名 (Case-insensitive)
        Mount Point:               /                                    # 加载系统根目录
        Capacity Consumed:         11213705216 B (11.2 GB)
        FileVault:                 Yes (Unlocked)                       #加密状态：已解锁
```

通过`diskutil apfs listusers $Volume_ID`，查看当前有效的用户密钥信息，即有几个 KEK 副本。

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

### 2. 签名系统卷（M1 CPU）

MacOS 10.15 Catalina 引入了 APFS Volume Group，系统卷和数据卷属于同一卷组，并且在 Finder 中被视为一个卷；引入了只读系统宗卷，这是一个专用于系统内容（通常名为“Macintosh HD”）的独立宗卷，默认不能写入数据，甚至 Apple 系统进程也不能。
MacOS 11 Big Sur 将只读系统宗卷升级为签名系统卷 (SSV，Sealed & Signed System Volume)，进一步增加了操作系统的签名保护，甚至现在启动系统的都不是真实的 System 卷宗，而是启动时创建的一个快照。
SSV 具有的内核机制会在运行时验证系统内容的完整性，并拒绝不含来自 Apple 的有效加密签名的任何代码和非代码数据。此外，还有一个附带的优势，在进行操作系统更新时如果发生意外无法执行， 无需重新安装即可恢复到旧系统版本。
> SSV 签名系统卷依赖于 SKP 密钥（Sealed Key Prtection，密封密钥保护，也称为操作系统绑定密钥），也就是依赖于 Apple 安全隔区硬件，因此 Intel CPU 不适用。

通过 `diskutil list`，可以看到物理硬盘`disk0`包含了3个容器：

- Apple_APFS_ISC 容器：ISC（iBoot System Container）容器负责在早期引导过程中支持 iBoot 固件，并为 M1 SoC 中的 Secure Enclave 提供可信存储。
    iSCPreboot 卷是指定的引导程序，空的 Recovery 卷用于恢复。xART 卷提供可信存储，Hardware 卷包含与硬件相关的文件。
- Apple_APFS_Recovery 容器：专用于提供 1TR，存储在其 Recovery 卷上。包括 iBoot 的第二部分以及 M1 的完整恢复模式所需的所有内容。
    该 Recovery 卷被指定用于恢复，但此该容器没有单独的引导程序卷。
- Apple_APFS 容器：M1 的引导容器 Apple_APFS 也与 Intel Mac 上的引导容器不太一样：一个细微但显著的区别是数据卷不是命名为“Macintosh HD - Data”，而是简单的“Data”。如果使用依赖于按名称查找数据卷的代码，则要重新检查它代码是否仍然有效。
    尽管此容器仍有一个 Recovery 卷，但该 Recovery 卷已经受到了一些限制，比如无法访问安全策略等，在引导到恢复模式时也没有使用该 Recovery 卷。

```console
sj@SunJiandeMacBook-Air ~ % diskutil list     
/dev/disk0 (internal, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *1.0 TB     disk0
   1:             Apple_APFS_ISC Container disk1         524.3 MB   disk0s1
   2:                 Apple_APFS Container disk3         994.7 GB   disk0s2
   3:        Apple_APFS_Recovery Container disk2         5.4 GB     disk0s3

/dev/disk3 (synthesized):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      APFS Container Scheme -                      +994.7 GB   disk3
                                 Physical Store disk0s2
   1:                APFS Volume Untitled - Data         397.7 GB   disk3s1
   2:                APFS Volume Untitled                12.0 GB    disk3s3
   3:              APFS Snapshot com.apple.os.update-... 12.0 GB    disk3s3s1
   4:                APFS Volume Preboot                 10.0 GB    disk3s4
   5:                APFS Volume Recovery                1.7 GB     disk3s5
   6:                APFS Volume VM                      20.5 KB    disk3s6
```

![M1](m1.jpg)

通过`diskutil apfs list`，可以发现增加了一个签名系统卷`disk3-s3`，隐藏了一个系统卷`disks3-s2`，实际指向了 SSV 的快照`disks3-s3-s1`。

```console
sj@SunJiandeMacBook-Air ~ % diskutil apfs list
APFS Containers (3 found)
|
+-- Container disk3 F2E4F923-BAEF-4844-8720-A1E3C4A91D68
    ====================================================
    APFS Container Reference:     disk3
    Size (Capacity Ceiling):      994662584320 B (994.7 GB)
    Capacity In Use By Volumes:   422349295616 B (422.3 GB) (42.5% used)
    Capacity Not Allocated:       572313288704 B (572.3 GB) (57.5% free)
    |
    +-< Physical Store disk0s2 03B7C267-30AE-414D-93B3-82B5E6A7D794
    |   -----------------------------------------------------------
    |   APFS Physical Store Disk:   disk0s2
    |   Size:                       994662584320 B (994.7 GB)
    |
    +-> Volume disk3s1 4C4F95CE-27C8-4CC8-9287-0746D8B6F445
    |   ---------------------------------------------------
    |   APFS Volume Disk (Role):   disk3s1 (Data)
    |   Name:                      Untitled - Data (Case-insensitive)
    |   Mount Point:               /System/Volumes/Data
    |   Capacity Consumed:         397743120384 B (397.7 GB)
    |   Sealed:                    No
    |   FileVault:                 Yes (Unlocked)
    |
    +-> Volume disk3s3 2F5B627D-273D-43E1-B33D-11A51F2DD616
    |   ---------------------------------------------------
    |   APFS Volume Disk (Role):   disk3s3 (System)                         # SSV卷宗
    |   Name:                      Untitled (Case-insensitive)
    |   Mount Point:               /System/Volumes/Update/mnt1
    |   Capacity Consumed:         12000673792 B (12.0 GB)
    |   Sealed:                    Broken
    |   FileVault:                 Yes (Unlocked)
    |   Encrypted:                 No
    |   |
    |   Snapshot:                  888DEA8C-D791-4F5B-BC62-26E2D6A436E4     # 快照卷宗
    |   Snapshot Disk:             disk3s3s1
    |   Snapshot Mount Point:      /
    |   Snapshot Sealed:           Yes                                      # 认证通过
    |
    +-> Volume disk3s4 C828BF80-F6B2-4370-8C12-5DC67A144D2A
    |   ---------------------------------------------------
    |   APFS Volume Disk (Role):   disk3s4 (Preboot)
    |   Name:                      Preboot (Case-insensitive)
    |   Mount Point:               /System/Volumes/Preboot
    |   Capacity Consumed:         10008113152 B (10.0 GB)
    |   Sealed:                    No
    |   FileVault:                 No
    |
    +-> Volume disk3s5 48782923-A22F-45A8-A688-E4F72065E32B
    |   ---------------------------------------------------
    |   APFS Volume Disk (Role):   disk3s5 (Recovery)
    |   Name:                      Recovery (Case-insensitive)
    |   Mount Point:               Not Mounted
    |   Capacity Consumed:         1675984896 B (1.7 GB)
    |   Sealed:                    No
    |   FileVault:                 No
    |
    +-> Volume disk3s6 1793DE3F-A7B8-4103-A03A-91113BF324E7
        ---------------------------------------------------
        APFS Volume Disk (Role):   disk3s6 (VM)
        Name:                      VM (Case-insensitive)
        Mount Point:               /System/Volumes/VM
        Capacity Consumed:         20480 B (20.5 KB)
        Sealed:                    No
        FileVault:                 No
```

## 四、FileVault的密钥层级

对比 Data Protection 数据保护技术，FileVault 等价于 C 类，使用基于 AES-XTS 的 FDE 全盘加密模式保护卷宗数据。经过多年的技术演进，最新、最复杂的 Apple FileVault 密钥层级如下图。

![SKP](filevault-on.png)

- SKP（Sealed Key Prtection，密封密钥保护，也称为操作系统绑定密钥）：Apple 为其设计的 SoC 设备开发的，旨在确保其专用设备被拆解、或使用未认证的操作系统版本等场景下，无法获取加密材料，目的是为系统密钥提供独立于安全隔区的额外保护。
    注意！SKP 功能**不是由安全隔区提供的**，而是由位于更底层的硬件寄存器支持，实际上是基于安全隔区操作系统 sepOS 加载时的测量值。这也意味着，其仅在搭载 Apple 设计的 SoC 的设备上提供。
- xART（eXtended Anti-Replay Technology，扩展反重放技术）：一组为具有反重放功能 (基于物理储存架构) 的安全隔区提供加密且经认证的永久储存区的服务。
    xART 是 Apple 设计的第二代安全存储组件，增加了一个计数器加密箱，包括：1个128位盐，1个128位密码验证器，1个8位计数器，1个8位最大尝试值。
- KEK（Key Encryption Key，密钥保护密钥）：派生密钥，基于用户口令、SKP密钥和硬件密钥（基于UID派生）进行密钥扩展生成
- VEK（Volume Encryption key，卷宗保护密钥）：随机生成密钥，基于 KEK 和 硬件密钥进行包裹，并被 xART 保护，就是用于卷宗数据加密的 AES-XTS 的分组密钥 key1。

如上所述，搭载 Apple 芯片的 Mac 以及搭载 T2 芯片的 Mac 通过构建和管理密钥层级实施内部宗卷加密，基于芯片内建的硬件加密技术而构建。
在搭载 Apple 芯片的 Mac 以及搭载 T2 芯片的 Mac 上，所有文件保险箱密钥的处理都发生在安全隔区中；加密密钥绝不会直接透露给 Intel CPU。
所有 APFS 宗卷默认使用宗卷加密密钥创建。宗卷和元数据内容使用此宗卷加密密钥加密，此宗卷加密密钥使用类密钥封装。 文件保险箱启用时，类密钥受用户密码和硬件 UID 共同保护。
如果没有有效的登录凭证（User passcode）或加密恢复密钥（Recovery key），即使物理储存设备被移除并连接到其他电脑，内置 APFS 宗卷仍无法解密，以防止未经授权的访问。

删除宗卷时，其宗卷加密密钥由安全隔区安全删除，这有助于防止以后使用此密钥进行访问 (即使是通过安全隔区)。另外，所有宗卷加密密钥都使用媒介密钥封装。媒介密钥不提供额外的数据机密性，而是旨在启用快速安全的数据删除，如果缺少了它，则不可能进行解密。
在搭载 Apple 芯片的 Mac 和搭载 T2 芯片的 Mac 上，媒介密钥一定是由受安全隔区支持的技术来抹掉，例如远程 MDM（Mobile Device Management，远程-移动设备管理） 命令。以这种方式抹掉媒介密钥将导致宗卷因存在加密而不可访问。

可移除储存设备的加密不使用安全隔区的安全性功能，而是采用与基于 Intel 的 Mac (不搭载 T2 芯片) 相同的方式执行加密。

### 1. 版本演进

- 在 macOS 10.7 Lion 中，引入了 CoreStorage 管理卷，并规定 VEK 密钥的创建点是用户在 Mac 上**启用FileVault**的过程中。
- 在 macOS 10.13 High Sierra 中，引入了 AFPS 文件系统，VEK 密钥的创建点调整为：**用户创建过程中**、 设定首位用户的密码或 Mac 用户首次登录过程中。换句话说，无论用户是否启用 FileVault，卷宗数据都会被加密，其区别仅在于：
  - 如果没有启用 FileVault 功能，宗卷加密密钥仅由安全隔区中的硬件 UID 保护
         ![SKP](filevault-off.png)
  - 如果稍后启用了文件保险箱 (由于数据已加密，该过程可快速完成)，反重放机制会帮助阻止旧密钥 (仅基于硬件 UID) 被用于解密宗卷，然后宗卷将受用户密码和硬件 UID 共同保护
- 在 macOS 10.15 Catalina中，这是第一个仅支持 64 位应用程序的 macOS 版本！引入了 Bootstrap Token 功能，也就是为密钥层级增加了 SKP 保护层，并为后续签名系统卷宗 SSV 提供了技术基础。
- 在 macOS 11 Big Sur 中，系统宗卷通过签名系统宗卷 SSV 功能进行保护（实际上仅提供操作系统的快照），而数据宗卷仍通过加密进行保护。

### 2. KEK的多副本

Apple公司定义了几种获得 KEK 的方式：

- 用户口令（User password）：系统登录时用户输入口令。为保证必要的安全强度，系统实际使用的密钥必须经过密钥拉伸
- 个人恢复密钥（Personal recovery key）：在格式化驱动器时生成的，并由用户保存在纸上或打印输出，系统或厂商并不留存
- 机构恢复密钥（Institutional recovery key）：允许相应的公司获得 KEK，需要用户设置后生效，或者厂商偷偷强制设置！
- 远程恢复密钥（iCloud recovery key）：用户在Apple公司的远程支持下进行系统恢复

为支持不同的场景，keyBag 将同时存储多个 KEK 的副本，例如用户口令采用密钥包裹方式存储，而 iCloud 恢复密钥采用非对称的公钥加密存储，不同场景使用不同的包裹密钥，这也体现了多层级密钥管理的价值所在。

## 五、FileValut 2 的解密代码实例

根据[Infiltrate the Vault: Security Analysis and Decryption of Lion Full Disk Encryption - Omar Choudary](2012-374.pdf)，密钥层次如下图。
参考[FVDE工具包](https://github.com/libyal/libfvde/blob/main/documentation/FileVault%20Drive%20Encryption%20(FVDE).asciidoc)，可以查看实现代码。

![密钥层次架构](arch.png)

### 1. CoreStorage Header

在 CoreStorage 加密卷的 Header，存储了核心的加密信息，包括：卷头部签名、块大小、卷大小、元数据大小、第一个元数据块块号、第二个元数据块块号、第三个元数据块块号、第四个元数据块块号、加密方法、Physical Volume UUID（用于解密加密的密钥文件）、Logiccal Volume Group UUID 等。
![CS](CS-Header.png)

### 2. EncryptedRoot.plist

在恢复数据卷 Recovery HD，有一个加密文件包含了提取 VMK 所需的全部信息，路径是：`com.apple.boot.X/System/Library/Caches/com.apple.corestorage/EncryptedRoot.plist.wipekey`
这个文件本身也是基于 AES-XTS 加密的，分组密钥就是 CoreStorage 头部存储的`AES-XTS key1`，可调整密钥是 128 位的全零，解密成功后的内容实例：

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

### 3. Volume Master Key

解密后的 plist 文件是一个 XML 文件，其中的关键字段如下，注意实际存储格式是 base64 编码：

#### PassphraseWrappedKEKStruct

KEK（Key Encryption Key，密钥保护密钥）的构造体：AES包裹的KEK，PBKDF2算法的盐。
包含了 2 个 284 位 的数据块，分别用于 recovery password 和 user password。

![CS3](CS3.png)

> 注意！MacOS 的每个用户都有其自己关联的 PassphraseWrappedKEKStruct
> APFS 称之为 VEK（Volume Encryption Key）

#### KEKWrappedVolumeKeyStruct

VMK（Volume Master Key，卷宗主密钥）的构造体。
数组有多个成员，其中标记为`AES-XTS`的就是 KEK 包裹的 VMK。
![CS2](CS2.png)

#### KeyWrappedKEK

如果用户开启了 icloud 远程备份，plist 文件将包含该字段，其数据块结构如下：
![CS4](CS4.png)
> 注意！icloud 恢复是另一个基于非对称密钥算法 RSA 封装的 KEK

#### 伪代码实例

无论基于 recovery password，还是 user password，都需要通过 PBKDF2 算法进行密钥拉伸。
> 注意！recovery password 是字符串格式，包括数字之间的破折号。

![VMK](VMK.png)

``` c
p = get_user_password()
salt = get_salt_from_PassphraseWrappedKEK()
iterations = 41000
pk = pbkdf2(p, salt, iterations, HMAC-SHA256)
kek_wrapped = get_kek_from_PassphraseWrappedKEK()
kek = aes_unwrap(kek_wrapped, pk)
vmk_wrapped = get_vmk_from_KEKWrappedVolumeKey()
vmk = aes_unwrap(vmk_wrapped, kek)
```

### 4. Volume Tweak Key

用户数据 AES-XTS 加密模式的 tweak key 也是128位，构造方式是：$trunc_{128}(SHA256(VolumeMasterKey || LogicVolumeFamilyIdentifier))$

上个流程已经找到 VMK，那么如何找到 LV Family UUID 呢？

- 在 CoreStorage Header 中，第104字节提供了字节偏移量，就是 Disk Label Metadata 的存储位置
- 在 Disk Label 中，第220字节提供了字节偏移量，就是 Encrypted Metadata 的存储位置
    即：$offset = DiskLabel[DiskLabel[220] +32]$
- Encrypted Metadata 基于 AES-XTS 加密，分组密钥也是 CoreStorage Header 的 AES-XTS Key1，可调整密钥是 CoreStorage Header 的 PV UUID
- 解密后的 Encrypted Metadata，第280字节提供了字节偏移量，就是 XML Metadata 的存储位置
    ![CS6](CS6.png)
- 有3个 XML 文件，包含了许多UUID，其中第1和第3个 XML 文件包含了 Logical Volume Family UUID，就是下图中蓝色字段
    ![CS7](CS7.png)

---

## 参考文献

- [Legacy FileVault 漏洞报告](https://fahrplan.events.ccc.de/congress/2006/Fahrplan/attachments/1244-23C3VileFault.pdf)
- [MacOS X 的 Corestorage 逻辑卷组管理器 - Stephen Foskett](https://blog.fosketts.net/2011/08/04/mac-osx-lion-corestorage-volume-manager/)
- [MacOS 的安全和隐私指南](https://github.com/drduh/macOS-Security-and-Privacy-Guide/blob/master/README-cn.md)
- [Working with APFS Volume Groups](https://bombich.com/kb/ccc5/working-apfs-volume-groups)
- [APFS 科普贴](https://www.jianshu.com/p/c401d546cebf)
- [APFS Structure](https://www.ntfs.com/apfs-structure.htm)
- [M1 Macs radically change boot and recovery](https://eclecticlight.co/2021/01/14/m1-macs-radically-change-boot-and-recovery/)
- [Mac 迁移指南：换新机后的任务清单](https://sspai.com/post/64301)
- [FileVault Drive Encryption 代码库 - Github](https://github.com/libyal/libfvde/blob/main/documentation/FileVault%20Drive%20Encryption%20(FVDE).asciidoc)

### 文档下载

- [FileVault2加密分区离线解密技术及其取证应用 - 蓝朝祥](FileVault2加密分区离线解密技术及其取证应用_蓝朝祥.pdf)
- [Security Analysis and Decryption of FileVault 2 - Omar Choudary](slides_fv2_ifip_2013.pdf)
- [Infiltrate the Vault: Security Analysis and Decryption of Lion Full Disk Encryption - Omar Choudary](2012-374.pdf)
- [APFS技术白皮书](<Apple-File-System-Reference.pdf>)
