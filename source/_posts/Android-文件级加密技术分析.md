---
title: Android 文件级加密技术分析
date: 2024-05-03 09:56:21
tags:
---

Full-Disk Encryption（全盘加密）是一种单密钥的加密系统，有硬件、软件两种实现方案。

- 硬件方案：也称为 Self Encryption Drive（自加密硬盘），一般由存储器件厂商、安全厂商提供，出厂时内嵌预启动环境 bootLoader，上电后先运行 Pre-Boot 程序并验证 AK（Authentication Key），再进行正常的加载引导。请参见[Seagate 硬盘 的 NIST FIPS 140-2 认证报告](https://csrc.nist.gov/csrc/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp1299.pdf)。
- 软件方案：采用不加密boot分区的方式，一般在 initial ramdisk 集成 PBA 功能并完成验证。例如 Windows 的BitLocker、Apple 的 FileVault、以及 Linux 的 dm-crypt/LUKS。

File-Based Encryption 是一种多密钥的加密系统，又称 Filesystem-Level Encryption（文件系统加密）。相比于 FBE，第二个名字更能体现方案基于文件系统的技术特点。而其基于文件系统的特点，一方面决定了只能由软件实现，另一方面决定了各方案差异也主要在文件系统，有两种类型：

- Stackable cryptographic filesystem（堆叠加密文件系统）：新增一个加解密文件系统，堆叠在现有存储软件栈的某一层。例如 Linux 内核自 v2.6.19 开始支持，已很成熟稳定的 [eCryptfs](https://www.kernel.org/doc/html/v5.7/filesystems/ecryptfs.html) 方案，就是在 VFS -> Native FS 层之间加入新加解密文件系统支持。类似还有基于用户态文件系统 [FUSE - Filesystem in Userspace](https://zh.wikipedia.org/wiki/FUSE)的各种方案。
- Native filesystem with encryption（原生加密文件系统）：在现有文件系统中引入加解密功能。例如 Linux 内核自 v4.1 支持的 Ext4 文件系统加密，自 v4.2 支持的 [F2FS 文件系统](https://www.kernel.org/doc/html/v5.7/filesystems/f2fs.html)加密，自 v4.10 后支持的 UBIFS 文件系统加密。需要说明的是，内核中 Ext4、F2FS、ubifs 共用加解密功能模块，即内核 [fscrypt](https://www.kernel.org/doc/html/v4.18/filesystems/fscrypt.html) 特性。

和 FDE 方案相比，FBE 有几个显著的特点：

- 支持单独的目录或文件加密，方便灵活使用配置。只加密目标对象，不加密整个磁盘，降低了系统加解密负载开销。
- 支持不同目录/文件使用不同加密密钥。
- 加密目录和非加密目录并存（甚至一个加密目录中加密和非加密文件也可以并存）。加密目录文件的备份传输灵活方便。

## 一、概述

Android 设备可以访问和存储用户的个人隐私数据，如果设备一旦丢失，会极大的增加用户数据泄露的风险。从 Android 4.4 开始，Google 相继推出多种加密方案：

- Android 5.0 到 Android 9 支持 FDE 加密方案
- Android 7.0 及更高版本支持 FBE 加密方案
- Android 9 引入了 ME（Metadata Encryption，文件元数据加密），但需要硬件支持

为了安全地使用 AOSP 的 FBE 实现，设备需要满足以下依赖关系：

- 对 Ext4 加密或 F2FS 加密的内核支持
- 基于 1.0 或更高版本 HAL 的 Keymaster 支持，不支持 Keymaster 0.3
- 必须在可信执行环境 (TEE) 中实现 Keymaster/Keystore 和 Gatekeeper，以便为 DE 密钥提供保护，从而使未经授权的操作系统（刷写到设备上的定制操作系统）无法直接请求 DE 密钥
- 硬件信任根和启动时验证需要绑定到 Keymaster 初始化进程，以确保未经授权的操作系统无法获取 DE 密钥

## 二、技术架构

Android FBE 特性依赖 UFS（Universal Flash Storage，通用闪存存储） 和 TEE（Trusted Execution Environment，可行执行环境）。其中，TEE 提供基于硬件环境的 KMS（Key Management Service），该服务提供密钥操作，如密钥创建（Key Generation）、密钥派生（Key Derivation）、密钥编程（Key Programming）等；UFS 包括 UFS Core 和 UFS Device，UFS Controller 工作在 UFS Core 内部，用于接收来自 AP 的数据和命令。

![ARCH](arch.jpg)
ICE = Inline Crypto Engine（内联加密引擎），KSM = KeySlot Manager（密钥槽管理器）
TA = Trusted Application（可信应用），HAL = Hardware Abstract Layer（硬件抽象层）

## 三、实现方案

AOSP 实现基于 Linux 内核的 fscrypt 特性（受 ext4 和 f2fs 支持），并通常配置如下：

- 借助采用 XTS 模式的 AES-256 算法加密文件内容
- 借助采用 CBC-CTS 模式的 AES-256 算法加密文件名
- 对于部分 CPU 不支持 AES 指令的早期设备，AOSP 提供了 Adiantum 加密方法

Android 根据文件内容的私密性，把用户数据分区的存储位置划分安全等级，包括下几类：

- Unencrypted Storage：不加密的存储位置。iOS 没有该类型。
- System Device Encrypted (DE) Storage ：相当于 Class D。一般存储一些设备相关，Framework 相关等用户无关的数据。
- Device Encrypted (DE) Storage ：相当于 Class C，与用户相关的数据，安全性要求一般，在设备启动后以及用户解锁设备后都可以直接访问。
- Credential Encrypted (CE) Storage ：相当于 Class A，与用户密切相关的数据，安全性等级高，如果用户设置了锁屏密码，必须在用户解锁设备后这些存储位置的数据才可用。
- Android 没有相当于 Class B 的类型，可以用 DE 替代。

![A](android.jpg)

不同的 Storage 使用不同密钥。其中，System DE Storage 对应 SYSTEM_DE_KEY，User DE Storage 对应USER_DE_KEY，User CE Storage 对应 USER_CE_KEY。请参见[官方文档](https://source.android.com/docs/security/features/encryption/file-based?hl=zh-cn#key-storage-and-protection)

## 四、密钥层次结构

可以利用 HKDF 等 KDF（密钥派生函数）从其他密钥派生密钥，从而生成密钥层次结构。

### 不使用硬件封装密钥

![SW-key](fbe-key-hierarchy-standard.png)

FBE 类密钥是 Android 传递给 Linux 内核以解锁一组特定加密目录（例如针对特定 Android 用户的凭据加密存储空间）的原始加密密钥。（在此内核中，这种密钥称为 fscrypt 主密钥）。内核会根据该密钥派生以下子密钥：

- 密钥标识符。此标识符不用于加密，而是作为一个值用于标识保护特定文件或目录的密钥。
- 文件内容加密密钥
- 文件名加密密钥

### 使用硬件封装密钥

![HW-key](fbe-key-hierarchy-hw-wrapped.png)

与前一种情况相比，此密钥层次结构新增了一个额外级别，并且改变了文件内容加密密钥的位置。根节点仍代表由 Android 传递到 Linux 以解锁一组加密目录的密钥。但是，该密钥现在采用**临时封装**形式，并且必须传给**专用硬件**才能使用。该硬件必须实现两个获取一个临时封装密钥的接口：

- 第一个接口用于派生`inline_encryption_key`并将其直接编程到内嵌加密引擎的密钥槽。这样，软件无需访问原始密钥，即可加密/解密文件内容。在 Android 通用内核中，此接口与`blk_crypto_ll_ops::keyslot_program`操作相对应，此操作必须由存储驱动程序实现。
- 第二个接口用于派生并返回`sw_secret`（“软件 Secret”，在某些地方也称为“原始 Secret”），后者作为一个密钥由 Linux 用于为文件内容加密之外的所有加密派生子密钥。在 Android 通用内核中，此接口与 `blk_crypto_ll_ops::derive_sw_secret` 操作相对应，此操作必须由存储驱动程序实现。

如需从原始存储密钥派生`inline_encryption_key`和`sw_secret`，硬件必须使用强加密 KDF。此 KDF 必须遵循加密最佳实践；至少具有 256 位的安全性，也就是说，足以应对以后使用的任何算法。在派生每种类型的子密钥时，此 KDF 还必须使用不同的标签、上下文和/或应用特定信息字符串，以确保生成的子密钥经过加密隔离，也就是说，知道其中一个子密钥并不会泄露任何其他子密钥。原始存储密钥已是均匀随机密钥，因此不需要延伸密钥。

从技术上讲，可以使用任何满足安全要求的 KDF。但是，出于测试目的，需要在测试代码中重新实现相同的 KDF。目前，已审核并实现了一个 KDF；可以在[vts_kernel_encryption_test 的源代码](https://android.googlesource.com/platform/test/vts-testcase/kernel/+/4be1bd95bb9879375bd3f12089b0bc156529f19f/encryption/utils.cpp#402)中找到该 KDF。建议硬件使用此 KDF，此 KDF 使用 [NIST SP 800-108“计数器模式的 KDF”](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-108.pdf)，并将 AES-256-CMAC 作为 PRF。请注意，为确保兼容性，该算法的所有部分都必须完全相同，包括为每个子密钥选择的 KDF 上下文和标签。

---

## 附录：F2FS 文件系统

F2FS - Flash Friendly File System 是三星开发的、专门为闪存设备设计的开源 Flash 文件系统。

F2FS将整个卷切分成大量的 Segments，每个 Segment 的大小固定为 2MB。连续若干个 Segments 构成 Section，连续若干个 Section 构成 Zone。F2FS 文件系统将整个卷切分成 6 个区域，除了超级块（Superblock，简称SB）外，其余每个区域都包含多个 Segments，其结构如下图所示:
![F2FS](F2FS.jpg)

---

## 参考文献

- [详解Linux内核安全技术 - 磁盘加密技术概述](https://www.bilibili.com/read/cv24307831/)
- [Android系统安全技术 - FBE密钥框架和技术详解](https://blog.csdn.net/feelabclihu/article/details/131016357)
- [Android 檔案系統加密機制](https://www.kaotenforensic.com/android/android_encryption/)
- [Android 系統基本架構 - 開機流程與分區說明](https://www.kaotenforensic.com/android/booting-partitions/)
- [你的安卓手机究竟是FDE加密还是FBE加密？](https://page.om.qq.com/page/O3yauEIx2l-9WrUkHgQgRUBw0)

### 官方文档

- [FBE 文件级加密原理 - Android官方](https://source.android.com/docs/security/features/encryption/file-based?hl=zh-cn)
