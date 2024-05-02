---
title: Apple Data Protection的功能概览
date: 2024-01-27 14:34:06
tags:
---

长期以来 Apple 公司有两条数据安全的技术演进路线，iPhone 和 iPad 等智能终端设备使用称为**数据保护（Data Protection）**的文件加密方法，基于Intel的Mac设备通过**文件保险箱（FileVault）**的宗卷加密技术。

MacOS 和 iOS 都是基于 FreeBSD 发展而来，基础架构是一个多用户的操作系统，但 iPhone 等作为个人化（无需支持多用户）的终端设备，存储了通信录、照片等非常敏感的隐私数据，对数据保护的要求更高；其次，App Store 和 iCloud等云服务大量出现，在网络开放环境下既要支持便利的后台服务管理，又要解决数据迁移过程中的数据保护，对数据保护的技术方案提出了新要求；最后，跨平台的生态系统需要密切协同（例如多个不同设备之间需要共享邮箱账号和密码），需要解决服务协同与数据安全的矛盾。

数据保护的技术实现离不开关键的硬件支持！限于技术演进和商业利益，早期的 Mac 设备基于 Intel CPU，早期的 iPhone 也采用 ARM6 和 ARM7 CPU，其底层架构难以满足数据保护的技术需求，为此 Apple 先后推出了适配 iPhone 的 A 系列 CPU 和适配 Mac 设备的 T 系列增强型安全芯片，依托自主研发的 Secure Encalve 为数据保护技术提供底层硬件支持，同时结合操作系统内核技术和 APP 沙盒技术等，确保只有受信任的代码及App可以在设备上运行（典型成果就是流行一时的 iPhone 越狱已经成为历史！），并有效提升了安全启动链、系统安全性和App安全性功能。

FileVault 是一个单密钥的 FDE 系统，可以认为是 FBE 系统的早期简化版本，随着 M1 自研芯片和 APFS 文件系统的推出，上述两条技术路线正在逐步融合，搭载 Apple 芯片的 Mac 设备已经使用两者的混合模型。

根据Apple 官方文档，Data Protection 数据保护技术的设计目标包括：

- 在硬件被改动的情况下也能保护数据（替换组件/直接读取闪存等）
- 对抗离线攻击（物理方式获取闪存内的数据后用更强大的计算设备进行破解）
- 对不同的数据提供不同级别的保护（锁屏之后有些数据要保护，有些数据还需要访问）
- 需要时能够快速安全清除所有数据
- 考虑性能和功耗

## 一、文件数据保护

iOS 的所有用户数据文件都是加密存储的，Mac的情况复杂一些，文件系统加密是可选的，但USB外接存储就没有必要了。主要设计思想是：

- 每个文件创建时会生成一个`Per-file Key`，在文件写入闪存时通过硬件用`AES-XTS`加密
- `Per-file Key`存储在文件的元数据 (Metadata) 中，被`Class Key`和`Filesystem Key`加密
- `Class Key`和`Filesystem Key`都被硬件密钥`UID`保护
- `Class Key`还被用户密码`Passcode`保护

![key](KEY-ARCH.jpg)
> EMF Key 就是加密封装的 Filesystem Key，存储在闪存的可擦除区域，只允许安全隔区访问
> Class Key 通过多层加密封装，持久化存储在 keybag（系统密钥包）之中

这种设计方案的主要优势在于：

1. 实现了“一件一密”，每个文件都有独立的密钥`Per-file Key`，加密强度大大提高
2. `Class Key`实现对文件不同级别的保护（最高级别只有在设备解锁状态下才可用），一个密钥就可以解封对应级别所有文件的`Per-file Key`；当用户修改`Passcode`时，只需重置`Class Key`并对元数据中的文件密钥重新封装即可，原始文件无需重新加密，兼顾了安全性和灵活性
3. `Filesystem Key`在 iOS 安装时或每次设备数据被清除时生成，其目标不是保护数据的机密性，而是用于实现快速清除数据，即一旦`Filesystem Key`被删除，设备上所有文件将永久无法解密
4. 所有加密处理由 AES Engine 的硬件处理，性能有保障；同时所有密钥由 Secure Enclave 统一管理，不对外泄露，确保安全性并对上层应用透明
5. 文件数据有`Class Key`和`Filesystem Key`两层密钥，并最终由`UID`和`Passcode`提供保护，考虑到用户设置`Passcode`的强度不高，Apple还专门强化了安全隔区的反重放机制以提高安全性

> When a file is opened, its metadata is decrypted with the file system key, revealing the wrapped per-file key and a notation on which class protects it. The per-file (or per-extent) key is unwrapped with the class key and then supplied to the hardware AES Engine, which decrypts the file as it’s read from flash storage. All wrapped file key handling occurs in the Secure Enclave; the file key is never directly exposed to the Application Processor.

## 二、数据保护等级

设备中我们需要对不同的文件提供不同程度的保护。iOS 上每个文件创建时都会被指定一个级别，每个级别对应一个`Class Key`。

### A类：完全保护，Complete Protection

- 该类密钥通过从用户密码和设备 UID 派生的密钥得到保护。用户锁定设备后不久（如果 “需要密码” 设置为 “立即”，则为10秒钟），解密的类密钥会被丢弃，此类的所有数据都无法访问，除非用户再次输入密码或使用触控 ID 或面容 ID 解锁（登录）设备。
- 在 macOS 中，上一个用户退出登录不久后，解密的类密钥会被丢弃，此类的所有数据都无法访问，直到某位用户再次输入密码或使用触控 ID 登录设备。

### B类：未打开文件保护，Protected Unless Open

- 设备锁定或用户退出登录时，可能需要写入部分文件（如后台下载邮件附件）。此行为通过使用非对称椭圆曲线加密技术（基于 Curve25519 的 ECDH）实现。普通的文件独有密钥通过使用一次性迪菲-赫尔曼密钥交换协议（One-Pass Diffie-Hellman Key Agreement，如 NIST SP 800-56A 中所述）派生的密钥进行保护。
- 该协议的临时公钥与封装的文件独有密钥一起储存。 KDF 是串联密钥导出函数 (Approved Alternative 1)，如 NIST SP 800-56A 中 5.8.1 所述。AlgorithmID 已忽略。 PartyUInfo 和 PartyVInfo 分别是临时公钥和静态公钥。SHA256 被用作哈希函数。一旦文件关闭，文件独有密钥就会从内存中擦除。要再次打开该文件，系统会使用 “未打开文件的保护” 类的私钥和文件的临时公钥重新创建共享密钥，用来解开文件独有密钥的封装，然后用文件独有密钥来解密文件。
- 在 macOS 中，只要系统上的任何用户已登录或认证即可访问 NSFileProtectionCompleteUnlessOpen 的私有部分。

### C类：首次用户认证前保护，Protected Until First User Authentication

- 此类和 “全面保护” 类的行为方式相同，只不过在设备锁定或用户退出登录时已解密的类密钥不会从内存中删除。此类中的保护与桌面电脑全宗卷加密有类似的属性，可防止数据受到涉及重新启动的攻击。这是未分配给数据保护类的所有第三方 App 数据的默认类。
- 在 macOS 中，此类的作用类似于文件保险箱，且使用只要宗卷装载即可访问的宗卷密钥。

> 由于 MacOS 的 FileVault 是单密钥系统，实际上技术架构仅能支持 Class C

### D类：无保护，No Protection

- 此类密钥仅受 UID 的保护，并且存储在可擦除存储器中（即 DKey 密钥）。由于解密该类中的文件所需的所有密钥都储存在设备上，因此采用该类加密的唯一好处就是可以进行快速远程擦除。即使未向文件分配数据保护类，此文件仍会以加密形式储存 （就像 iOS 和 iPadOS 设备上的所有数据那样）。
- macOS 不支持该类型。

## 三、钥匙包（keyBag）数据保护

系统密钥包是一个加密的 `plist` 格式的二进制文件，存储了所有类密钥的数据，Apple 系统的数据解密实现方式见下图。

![Daemon](daemon.png)

有五种不同类型的钥匙包，但数据格式完全相同，包括：

### 1. 用户密钥包(User keyBag)

用户密钥包是设备常规操作中使用的封装类密钥的储存位置。例如，输入密码后，`NSFileProtectionComplete`会从用户密钥包中载入并解封。它是储存在“无保护”类中的二进制属性列表 (.plist) 文件。
对于搭载 A9 之前的 SoC 的设备，该 .plist 文件的内容**通过保存在可擦除存储器中的密钥（Bag1 Key）加密**。为了给密钥包提供前向安全性，用户每次更改密码时，系统都会擦除并重新生成此密钥。
对于搭载 A9 或后续型号 SoC 的设备，该 .plist 文件包含一个密钥，表示**密钥包储存在受反重放随机数（由安全隔区控制）保护的有锁储存库中**。

### 2. 设备密钥包 (Device keyBag)

设备密钥包用来储存用于涉及设备特定操作数据的封装类密钥。 配置为共用的 iPadOS 设备有时需要在用户登录前访问凭证，因此需要一个不受用户密码保护的密钥包。
iOS 和 iPadOS 不支持对用户独有的文件系统内容进行单独加密，这就意味着系统使用来自设备密钥包的类密钥，对文件独有密钥进行封装，而钥匙串则使用来自用户密钥包中的类密钥来保护用户钥匙串中的项目。
在配置为单用户使用 (默认配置) 的 iOS 和 iPadOS 设备中，**设备密钥包和用户密钥包是同一个**，并受用户的密码保护。

### 3. 备份密钥包（Backup keyBag）

备份密钥包在 “访达” (macOS 10.15 或更高版本) 或 iTunes (macOS 10.14 或更低版本) 进行加密备份时创建，并储存在设备被备份到的电脑中。
新密钥包是通过一组新密钥创建的，**备份的数据会使用这些新密钥重新加密**。 如前所述，不可迁移钥匙串项仍使用 UID 派生密钥封装，以使其可以恢复到最初备份它们的设备，但在其他设备上不可访问。
由于备份密钥包并未捆绑特定设备，理论上尝试在多台电脑上对备份密钥包并行展开暴力破解是可行的。
如果用户选择不加密备份，那么不管备份文件属于哪一种数据保护类，备份文件都不加密，但钥匙串仍使用 UID 派生密钥获得保护。这就是只有设置备份密码才能将钥匙串项迁移到新设备的原因。

### 4. 托管密钥包（Escrow keybag）

托管密钥包用于通过 USB 与 “访达” (macOS 10.15 或更高版本) 或 iTunes (macOS 10.14 或更低版 本) 进行同步， 还用于移动设备管理 (MDM)。 此密钥包允许 “访达” 或 iTunes 执行备份和同步， 而无需用户输入密码，它还允许 MDM 解决方案远程清除用户密码。它储存在用于与 “访达” 或 iTunes 进行同步的电脑上，或者在远程管理设备的 MDM 解决方案上。

托管密钥包改善了设备同步过程中的用户体验，期间可能需要访问所有类别的数据。当使用密码锁定的设备首次连接到 “访达” 或 iTunes 时，会提示用户输入密码。然后设备创建托管密钥包，其中包含的类密钥与设备上使用的完全相同，该密钥包由新生成的密钥进行保护。托管密钥包及用于保护它的密钥划分到设备和主机或服务器上，其数据以 “首次用户认证前保护” 类储存在设备上。这就是重新启动后，用户首次使用 “访达” 或 iTunes 进行备份之前必须输入设备密码的原因。

### 5.云备份密钥包（iCloud Backup keybag）

iCloud 云备份密钥包与备份密钥包类似。 该密钥包中的所有类密钥都是非对称的 (与 “未打开文件的保护”数据保护类一样， 使用 Curve25519)。
非对称密钥包还可用于 “iCloud 钥匙串” 钥匙串恢复中的备份。

## 四、钥匙串 (Key Chain)数据保护

许多 App 都需要处理密码和其他一些简短但比较敏感的数据，如密钥和登录令牌。钥匙串提供了储存这些项的安全方式。不同的 Apple 操作系统采用不同机制实施与各钥匙串保护类关联的保障。
在 macOS (包括搭 载 Apple 芯片的 Mac) 中，数据保护不直接用于实施此类保障。

钥匙串项使用两种不同的 AES-256-GCM 密钥加密 : 表格密钥 (元数据) 和行独有密钥 (私密密钥)。钥匙串元数据 (除 kSecValue 外的所有属性) 使用元数据密钥加密以加速搜索，私密值 (kSecValueData) 使用私密密钥进行加密。元数据密钥受安全隔区保护，但会缓存在应用程序处理器中以便进行钥匙串快速查询。私密密钥则始终需要通过安全隔区进行往返处理。

> 元数据密钥就是 EMF key，私密密钥根据 WARP 类型不同采用 Dkey 或 Dkey + Passcode Key

钥匙串以储存在文件系统中的`SQLite`数据库的形式实现， 而且数据库只有一个`securityd`监控程序决定每个进程或 App 可以访问哪些钥匙串项。钥匙串访问 API 将生成对监控程序的调用，从而查询 App 的 “keychain-access-groups”、 “application-identifier” 和 “application-group” 权限。访问组允许在 App 之间共享钥匙串项，而非将访问权限限制于单个进程。

### 1. 类型定义

|Keychain的数据保护类型|File的数据保护类型|适用场景|
|:---:|:---:|:---:|
|kSecAttrAccessibleWhenUnlocked|NSFileProtectionComplete|未锁定状态|
|N/A |NSFileProtectionCompleteUnlessOpen|锁定状态|
|kSecAttrAccessibleAfterFirstUnlock|NSFileProtectionCompleteUntilFirstUserAuthentication |首次解锁后|
|kSecAttrAccessibleAlways|NSFileProtectionNone|始终|

三个钥匙串类都有对应的`ThisDeviceOnly`项目，后者在备份期间从设备拷贝时始终通过 UID 加以保护，因此如果恢复至其他设备将无法使用，例如 VPN 证书不适合迁移至另一台设备。

文件的 Class B 使用了非对称加密算法，钥匙串不提供相应的 Class key。如果 APP 确实存在后台更新的需求，官方建议使用`kSecAttrAccessibleAfterFirstUnlock`，即对应 Class C。

还有一个特殊类型`kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly`，其仅存在于系统密钥包，行为方式与`kSecAttrAccessibleWhenUnlocked`相同。一旦用户密码被移除或重设，该类密钥便会丢弃，且不会备份、不同步到 iCloud 钥匙串、不包括在托管密钥包。

从 iOS 14 开始，`kSecAttrAccessibleAlways`和`kSecAttrAccessibleAlwaysThisDeviceOnly`被弃用！请参考[Apple开发手册](https://developer.apple.com/documentation/security/ksecattraccessiblealways)，

### 2. 典型场景

Apple 根据所保护信息的类型以及 iOS 和 iPadOS 需要这些信息的时间来选择钥匙串类，妥善平衡了安全性和可用性。例如，VPN 证书必须始终可用，这样设备才能保持连接，但它归类为 “不可迁移”， 因此不能将其移至另一台设备.

|Item|                                Accessibility|
|:-:|:-:|
|Wi-Fi passwords                   |Always|
|IMAP/POP/SMTP accounts             |AfterFirstUnlock|
|Exchange accounts              |Always|
|VPN                   |Always|
|LDAP/CalDAV/CardDAV accounts       |Always|
|iTunes backup password             |WhenUnlockedThisDeviceOnly|
|Device certificate & private key   |AlwaysThisDeviceOnly|

## 五、简要分析

### 1. 为什么用户修改 passcode 需要较长的处理时间？

用户文件的内容是基于 per-file key（或者VEK）加密的，修改 passcode 无需重新加密内容！但是，此时作为 KEK 的 passcode key 发生变化，需要重新计算并存储文件系统元数据中的 per-file key 密文和相应的 Class key。

> 文件的内容可能使用文件独有(或范围独有)的一个或多个密钥进行加密，密钥使用类密钥封装并储存在文件的元数据中，文件元数据又使用文件系统密钥进行加密。类密钥通过硬件 UID 获得保护，而某些类的类密钥则通过用户密码获得保护。此层次结构既可提供灵活性，又可保证性能。例如，更改文件的类只需重新封装其文件独有密钥，更改密码只需重新封装类密钥。

实际测试中，iPhone 或 Macbook 修改 passscode 需要大约 15 分钟处理时间，与重新加密数据文件对比，这个代价是值得的，体现了多层次密钥架构的优势。

### 2. UUID 和 UDID 的区别是什么？

简单说，UUID 是基于应用的唯一标识符，UDID 是基于设备的唯一标识符（注意区别于 UID）。

UDID (Unique Device Identifier，唯一设备标识符) 是一个由 40 个字符组成的十六进制序列，用于唯一标识一台苹果设备。它相当于设备的指纹，使得软件开发者和服务提供商能够提供个性化的设备识别服务。
UDID 可以被设备用户查询，例如 Mac 笔记本用户可以在`系统信息-硬件概览`中查看硬件 UDID。
![UDID](UDID.png)

构造算法：`UDID = SHA1(序列号 + ECID + Lowcase (WiFi MAC地址) + Lowcase(蓝牙 MAC地址))`
UDID 由于其唯一性，如果被泄露，恶意方可能会跟踪用户的活动或跨应用行为。因此，保护 UDID 以及使用它时要遵循隐私法和透明度原则是必要的。

UUID（Universally Unique Identifier，通用唯一标识符）是一个由 RFC 4122 定义的 128 位序列（ 32 个字符的十六进制），基于设备上的某个特定应用的数据项定义。只要用户没有完全删除应用，那么此标识符将在应用启动之间保留，并且至少允许您在设备上使用特定应用来识别同一数据项。
UUID 保证跨空间和时间的唯一性，但是如果用户完全删除然后重新安装应用程序，则 ID 将被更改。

### 3. 数据保护概览的解读

数据保护通过构建和管理密钥层级来实施，并建立在 Apple 设备内建的硬件加密技术基础上。它通过将某个类分配给每个文件来实现对文件的逐个控制； 可访问性根据该类密钥是否已解锁确定。APFS（Apple 文件系统）使文件系统可进一步以各个范围为基础对密钥进行细分（文件的各个部分可以拥有不同的密钥）。

每次在数据宗卷中创建文件时，数据保护都会创建一个新的 256 位**文件独有密钥（per-file key）**，并将其提供给硬件 AES 引擎，此引擎会使用该密钥在文件写入闪存时对其进行加密。在搭载 A14 和 M1 的设备上，加密在 XTS 模式中使用 AES-256，其中 256 位文件独有密钥通过密钥派生功能[NIST Special Publication 800-108](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-108r1.pdf) 派生出一个 256 位 tweak 密钥和一个 256 位 cipher 密钥。采用 A9 到 A13、 S5 和 S6 的每一代硬件在 XTS 模式中使用 AES-128，其中 256 位文件独有密钥会被拆分，以提供一个 128 位 tweak 密钥和一个 128 位 cipher 密钥。

> 对于采用 FBE 加密模式的 iOS 设备，APFS 的文件系统元数据定义了`CRYPTO_STATE`记录类型，其中`state.persistent_class`字段的描述为：The protection class associated with the key；`state.persistent_key`字段的描述为：The wrapped key data.
> 随着 AES-128 升级为 AES-256，per-file key 的存储长度发生变化，为了避免修改 AFPS 数据结构，Apple 修改了 cipher key 的管理方式

在搭载 Apple 芯片的 Mac 上，数据保护默认为 C 类（请参阅数据保护类），但使用**宗卷密钥（Volume Encryption Key，VEK）**，而非范围独有密钥或文件独有密钥，可为用户数据高效重建 FileVault 安全模型。用户仍须选择使用 FileVault，以获得加密密钥层级搭配用户密码的全面保护。开发者也可以选择使用文件独有密钥或范围独有密钥的更高保护类。

> 对于采用 FDE 加密模式的 MacOS，Ciper key 都是 VEK，因此 AFPS 不包含`CRYPTO_STATE`记录类型，而是在`FILE_EXTENT`记录类型的`crypto_id`字段存储了 Tweak key

在支持数据保护的 Apple 设备上，每个文件通过唯一的**文件独有密钥（per-file key）或范围独有密钥（per-extent key）**进行保护。该密钥使用 NIST AED 密钥封装算法封装，之后会进一步使用多个**类密钥（Class key）**中的一个进行封装，具体取决于计划如何访问该文件。 随后封装的文件独有密钥储存在文件的元数据中。

使用 APFS 格式运行的设备可能支持文件克隆（使用写入时拷贝技术的零损耗拷贝）。如果文件被克隆，克隆的每一半都会得到一个新的密钥以接受传入的数据写入，这样新数据会使用新密钥写入媒介。久而久之，文件可能会由不同的范围（或片段）组成，每个映射到不同的密钥。但是，组成文件的所有范围受同一类密钥保护。

> APFS 采用 Copy On Write 的快照模式（快照卷存储原始数据，源卷存放更新数据），如果后续某个副本的部分数据被修改，可能造成一个文件被分成多个 Extent，即一个文件系统节点可能包含多个`FILE_EXTENT`记录，每条记录的`crypto_id`字段各不相同

当打开一个文件时，系统会使用**文件系统密钥（Volume Key，就是 EMF 或 LwVm）**解密文件的元数据，以显露出封装的文件独有密钥和表示它受哪个类保护的记号。文件独有或范围独有密钥使用类密钥解封，然后提供给硬件 AES 引擎，该引擎会在从闪存中读取文件时对文件进行解密。所有封装文件密钥的处理发生在安全隔区中；文件密钥绝不会直接透露给应用程序处理器。启动时，安全隔区与 AES 引擎协商得到一个临时密钥。当安全隔区解开文件密钥时，它们又通过该临时密钥再次封装，然后发送回应用程序处理器。

> SEP 启动时生成临时封装密钥，通过专用线路传送给 AES 引擎，内存掉电时被清除

数据宗卷文件系统中所有文件的元数据都使用**随机宗卷密钥（就是上述的 EMF）**进行加密，该密钥在首次安装操作系统或用户擦除设备时创建。此密钥由**密钥封装密钥（就是 Key 0x89B，由 UID 固定衍生）**加密和封装，密钥封装密钥由安全隔区长期储存，只在安全隔区中可见。每次用户抹掉设备时，它都会发生变化。在 A9（及后续型号）SoC 上，安全隔区依靠由反重放系统支持的熵来实现可擦除性，以及保护其他资源中的密钥封装密钥。有关更多信息，请参阅**安全非易失性存储器（Secure Nonvolatile Storage）**。

正如文件独有密钥或范围独有密钥一样，数据宗卷的元数据密钥绝不会直接透露给应用程序处理器；相反，*安全隔区会提供一个临时的启动独有的版本*。储存后，加密的文件系统密钥还会使用储存在可擦除存储器中的“**可擦除密钥（即 BAG1，A9之前的Soc设备适用）**” 封装或者使用受安全隔区反重放机制保护的**媒介密钥封装密钥（即 media key，A9之后的Soc设备适用）**进行封装。此密钥不会提供数据的额外机密性。相反，它可以根据需要快速抹掉（由用户使用 “抹掉所有内容和设置” 选项来抹掉，或者由用户或管理员通过从移动设备管理 (MDM) 解决方案、Microsoft Exchange ActiveSync 或 iCloud 发出远程擦除命令来抹掉）。以这种方式抹掉密钥将导致所有文件因存在加密而不可访问。

文件的内容可能使用文件独有（或范围独有）的一个或多个密钥进行加密，密钥使用类密钥封装并储存在文件的元数据中，文件元数据又使用文件系统密钥进行加密。类密钥通过硬件 UID 获得保护，而某些类的类密钥则通过用户密码获得保护。此层次结构既可提供灵活性，又可保证性能。例如，更改文件的类只需重新封装其文件独有密钥，更改密码只需重新封装类密钥。

---

## 附录：名词解释

### 功能组件类

- Effaceable Storage（可擦除存储器）：NAND 存储器中一个用于储存加密密钥的专用区域，可被直接寻址和安全擦除。尽管当攻击者实际占有设备时，可擦除存储器无法提供保护，但其中存储的密钥可用作密钥层级的一部分，用于实现快速擦除和前向安全性。
- Secure Storage Component（安全储存组件）：一个芯片，设计为使用不可更改的 RO 代码、硬件随机数生成器、加密引擎和物理篡改检测。在支持的设备上，安全隔区与用于储存反重放随机数的安全储存组件配对。为了读取和更新随机数，安全隔区和储存芯片采用安全协议来帮助确保对随机数的排他访问。此技术已更迭多代，提供了不同的安全性保证。
- keybag（密钥包）：一种用于储存一组类密钥的数据结构。每种类型(用户、设备、系统、备份、托管或iCloud云备份)的格式都相同。
  - 标头：版本(在 iOS 12 或更高版本中设为 4 )；类型(系统、备份、托管或 iCloud 云备份)；密钥包 UUID；HMAC(若密钥包已签名)；用于封装类密钥的方法：配合盐和迭代计数使用 Tangling 及 UID 或 PBKDF2。
  - 类密钥列表：密钥 UUID；类(哪个文件或钥匙串数据保护类)；封装类型(仅 UID 派生密钥；UID 派生密钥和密码派生 密钥)；封装的类密钥；非对称类的公钥。
- keychain（钥匙串）： 一种基础架构和一组 API、Apple 操作系统和第三方 App，用来储存和检索密码、密钥及其他敏感凭证。
- Boot ROM：设备的处理器在首次启动时所执行的第一个代码。作为处理器不可分割的一部分，Apple或攻击者均无法修改。
- sepOS：安全隔区固件，基于 Apple 定制版本的 L4 微内核。
- XNU（X is Not Unix）：Apple 操作系统中央的内核。默认为受信任状态，并强制执行代码签名、沙盒化、授权核对和地址空间布局 随机化 (ASLR) 等安全措施。

### 实体密钥类

- UID：一个 256 位的 AES 密钥，在设备制造过程中刻录在每个处理器上。这种密钥无法由固件或软件读取，只能由处理器的硬件 AES 引擎使用。若要获取实际密钥，攻击者必须对处理器的芯片发起极为复杂且代价高昂的物理攻击。UID 与设备上的任何其他标识符均无关，包括但不限于 UDID。
- ECID（集成电路 ID）：每台 iOS 和 iPadOS 设备上的处理器所独有的一个 64 位标识符。当在一台设备上接通电话 时，该设备通过低功耗蓝牙 (BLE) 4.0 进行短暂广播，使附近的 iCloud 配对设备停止响铃。广播的字节使用与“接力” 广播相同的方法来加密。作为个性化流程的一部分，此标识符不被视为机密。
- Media key（媒介密钥）：加密密钥层次的一部分，可帮助实现安全的立即擦除。
  - 在 iOS、iPadOS、Apple tvOS 和 watchOS 中，媒介密钥会封装数据宗卷上的元数据(因此，没有媒介密钥便无法访问所有文件独有密钥，也就无法访问受数据保护加密方法所保护的文件)。
  - 在 macOS 中，媒介密钥会封装文件保险箱所保护宗卷上的密钥材料、所有元数据和数据。
- per-file key（文件独有密钥）：数据保护用于在文件系统上加密文件的密钥。文件独有密钥使用类密钥封装，储存在文件的元数据中。
- filesystem key（文件系统密钥）：用于加密每个文件的元数据的密钥，包括其类密钥。存储在可擦除存储器中，用于实现快速擦除，并非用于保密目的。
- Passcode-derived key（密码派生密钥，PDK）：用户密码与长期 SKP 密钥和安全隔区的 UID 配合使用，由此派生加密密钥。

### 软件算法类

- key wrapping（密钥封装/包裹）：使用一个密钥来加密另一个密钥。iOS 和 iPadOS 根据 RFC 3394 使用 NIST AES 密钥封装。
- Tangling（密钥缠绕）：用户密码转换为密钥并使用设备的 UID 加强的过程。此过程帮助确保暴力攻击只能在特定设备上执行， 因此可限制攻击的频度且避免多部设备同时遭到攻击。Tangling 算法是 PBKDF2。这种算法为每次迭代使用加入 设备 UID 的 AES 密钥作为伪随机函数 (PRF)。
- xART（eXtended Anti-Replay Technology，反重放技术）：一组为具有反重放功能(基于物理储存架构)的安全隔区提供加密且经认证的 永久储存区的服务。
- Sealed Key Protection（密封密钥保护，SKP）：数据保护中的一种技术，其使用系统软件的测量值和仅在硬件中可用的密钥来保护(或密封)加密密钥。

---

## Apple官方文档

- [Apple 平台安全白皮书 - 2022年英文版](apple-platform-security-guide.pdf)
- [Apple 平台安全白皮书 - 2021年中文版](apple平台安全白皮书-2021中文版.pdf)
- [iOS 安全白皮书 - 2018年英文版](iOS安全白皮书-2018英文版.pdf)
- [Apple T2 安全芯片概览](Apple_T2_Security_Chip_Overview.pdf)
- [APFS 文件系统参考手册](Apple-File-System-Reference.pdf)
