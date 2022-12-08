---
title: Apple数据保护技术专题之一：基础架构
date: 2022-11-13 14:34:06
tags:
---

长期以来 Apple 公司有两条数据安全的技术演进路线，iPhone 和 iPad 等智能终端设备使用称为**数据保护（Data Protection）**的文件加密方法，基于Intel的Mac设备通过**文件保险箱（FileVault）**的宗卷加密技术。

MacOS 和 iOS 都是基于 FreeBSD 发展而来，基础架构是一个多用户的操作系统，但 iPhone 等作为个人化（无需支持多用户）的终端设备，存储了通信录、照片等非常敏感的隐私数据，对数据保护的要求更高；其次，App Store 和 iCloud等云服务大量出现，在网络开放环境下既要支持便利的后台服务管理，又要解决数据迁移过程中的数据保护，对数据保护的技术方案提出了新要求；最后，跨平台的生态系统需要密切协同（例如多个不同设备之间需要共享邮箱账号和密码），需要解决服务协同与数据安全的矛盾。

数据保护的技术实现离不开关键的硬件支持！限于技术演进和商业利益，早期的 Mac 设备基于 Intel CPU，早期的 iPhone 也采用 ARM6 和 ARM7 CPU，其底层架构难以满足数据保护的技术需求，为此 Apple 先后推出了适配 iPhone 的 A 系列 CPU 和适配 Mac 设备的 T 系列增强型安全芯片，依托自主研发的 Secure Encalve 为数据保护技术提供底层硬件支持，同时结合操作系统内核技术和 APP 沙盒技术等，确保只有受信任的代码及App可以在设备上运行（典型成果就是流行一时的 iPhone 越狱已经成为历史！），并有效提升了安全启动链、系统安全性和App安全性功能。

随着 M1 自研芯片的推出，上述两条技术路线正在逐步融合，搭载 Apple 芯片的 Mac 设备已经使用两者的混合模型，相信未来 FileVault 技术将逐步退出，因此本文主要研究Data Protection 技术。根据Apple 官方文档，其主要设计目标包括：

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

### D类：无保护，No Protection

- 此类密钥仅受 UID 的保护，并且存储在可擦除存储器中（即 DKey 密钥）。由于解密该类中的文件所需的所有密钥都储存在设备上，因此采用该类加密的唯一好处就是可以进行快速远程擦除。即使未向文件分配数据保护类，此文件仍会以加密形式储存 （就像 iOS 和 iPadOS 设备上的所有数据那样）。
- macOS 不支持该类型。

> The contents of a file may be encrypted with one or more per-file (or per-extent) keys that are wrapped with a class key and stored in a file‘s metadata, which in turn is encrypted with the file system key. The class key is protected with the hardware UID and, for some classes, the user’s passcode. This hierarchy provides both flexibility and performance. For example, **changing a file‘s class only requires rewrapping its per-file key, and a change of passcode just rewraps the class key**。

## 三、钥匙包（keyBag）数据保护

系统密钥包是一个加密的 `plist` 格式的二进制文件，存储了所有类密钥的数据，Apple 系统的数据解密实现方式见下图。

![Daemon](daemon.png)

有五种不同类型的钥匙包，但数据格式完全相同，包括：

### 1. 用户密钥包(User keyBag)

用户密钥包是设备常规操作中使用的封装类密钥的储存位置。例如，输入密码后，`NSFileProtectionComplete`会从用户密钥包中载入并解封。它是储存在“无保护”类中的二进制属性列表 (.plist) 文件。
对于搭载 A9 之前的 SoC 的设备，该 .plist 文件的内容通过保存在可擦除存储器中的密钥（Bag1 Key）加密。为了给密钥包提供前向安全性，用户每次更改密码时，系统都会擦除并重新生成此密钥。
对于搭载 A9 或后续型号 SoC 的设备，该 .plist 文件包含一个密钥，表示密钥包储存在受反重放随机数（由安全隔区控制）保护的有锁储存库中。

### 2. 设备密钥包 (Device keyBag)

设备密钥包用来储存用于涉及设备特定操作数据的封装类密钥。 配置为共用的 iPadOS 设备有时需要在用户登录前访问凭证，因此需要一个不受用户密码保护的密钥包。
iOS 和 iPadOS 不支持对用户独有的文件系统内容进行单独加密，这就意味着系统使用来自设备密钥包的类密钥，对文件独有密钥进行封装，而钥匙串则使用来自用户密钥包中的类密钥来保护用户钥匙串中的项目。
在配置为单用户使用 (默认配置) 的 iOS 和 iPadOS 设备中， 设备密钥包和用户密钥包是同一个， 并受用户的密码保护。

### 3. 备份密钥包（Backup keyBag）

备份密钥包在 “访达” (macOS 10.15 或更高版本) 或 iTunes (macOS 10.14 或更低版本) 进行加密备份时创建，并储存在设备被备份到的电脑中。
新密钥包是通过一组新密钥创建的，备份的数据会使用这些新密钥重新加密。 如前所述，不可迁移钥匙串项仍使用 UID 派生密钥封装，以使其可以恢复到最初备份它们的设备，但在其他设备上不可访问。
由于备份密钥包并未捆绑特定设备， 理论上尝试在多台电脑上对备份密钥包并行展开暴力破解是可行的。 
如果用户选择不加密备份， 那么不管备份文件属于哪一种数据保护类， 备份文件都不加密，但钥匙串仍使用 UID 派生密钥获得保护。这就是只有设置备份密码才能将钥匙串项迁移到新设备的原因。

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

> 元数据密钥就是 FileSystem Key（EMF），私密密钥就是 Passcode Key

钥匙串以储存在文件系统中的`SQLite`数据库的形式实现， 而且数据库只有一个`securityd`监控程序决定每个进程或 App 可以访问哪些钥匙串项。钥匙串访问 API 将生成对监控程序的调用，从而查询 App 的 “keychain-access-groups”、 “application-identifier” 和 “application-group” 权限。访问组允许在 App 之间共享钥匙串项，而非将访问权限限制于单个进程。

### 1. 类型定义

根据 Apple 最新的 [KeyChain 的开发技术文档](https://developer.apple.com/documentation/security/ksecattraccessiblewhenunlockedthisdeviceonly)，当前提供如下类型：

对应文件保护的类型定义：

- kSecAttrAccessibleAfterFirstUnlock: 首次解锁后，对应文件保护的 Class C
- kSecAttrAccessibleWhenUnlocked: 未锁定状态下，对应文件保护的 Class A
- kSecAttrAccessibleAlways: **iOS 13 已弃用**。始终，对应文件保护的 Class D

> 使用后台刷新服务的 App 可通过 kSecAttrAccessibleAfterFirstUnlock 访问需要的钥匙串项

钥匙串类都有对应的 “仅限本设备” 项目，后者在备份期间从设备拷贝时始终通过 UID 加以保护，如果恢复（迁移）至其他设备将无法使用，例如 VPN 证书。

- kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly: 仅限本设备，首次解锁后
- kSecAttrAccessibleWhenUnlockedThisDeviceOnly: 仅限本设备，未锁定状态下
- kSecAttrAccessibleAlwaysThisDeviceOnly: **iOS 13 已弃用**。仅限本设备，始终

还有一个仅存在于系统密钥包的特殊类型，一旦用户密码被移除或重设，该类密钥便会丢弃，且不会备份、不同步到 iCloud 钥匙串、不包括在托管密钥包。

- kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly：仅当设备配置了密码时可用，行为方式与“未锁定状态下”相同

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

## 五、文件保管箱（FileVault）

> On Apple devices that support Data Protection, the key encryption key (KEK) is protected (or sealed) with measurements of the software on the system, as well as being tied to the UID available only from the Secure Enclave.
> On a Mac with Apple silicon, the protection of the KEK is further strengthened by incorporating information about security policy on the system, because macOS supports critical security policy changes (for example, disabling secure boot or SIP) that are unsupported on other platforms.
> On a Mac with Apple silicon, this protection encompass FileVault keys, because FileVault is implemented using Data Protection (Class C).
> The key that results from entangling the user password, long-term SKP key, and Hardware key 1 (the UID from Secure Enclave) is called the password-derived key.
> This key is used to protect the user keybag (on all supported platforms) and **KEK (in macOS only)**, and then enable biometric unlock or auto unlock with other devices such as Apple Watch.

MacOS 使用文件保险箱（FileVault）技术提供数据保护功能，使用 AES-XTS 数据加密算法保护内部和可移除储存设备上的完整宗卷。

> xART Key 是反重放密钥，SKP、KEK ？？？

数据宗卷文件系统中所有文件的元数据都使用随机宗卷密钥（`Volume Key`）进行加密，该密钥在首次安装操作系统或用户擦除设备时创建。此密钥由密钥封装密钥（`Key Encryption Key`）加密和封装，**密钥封装密钥由安全隔区长期储存，只在安全隔区中可见**。每次用户抹掉设备时，它都会发生变化。在 A9（及后续型号）SoC 上，安全隔区依靠由反重放系统支持的熵来实现可擦除性，以及保护其他资源中的密钥封装密钥。

> Volume Key 负责加密 volume 和文件 metadata，其作用与 iOS 的 FileSystem Key 相同，但加密和存储方式存在显著差异：
> iOS 中，FileSystem Key 的密文是 EMF，持久化存储在 NAND 可擦除区域；而 MacOS 将

在搭载 Apple 芯片的 Mac 上，数据保护默认为 C 类，但**使用宗卷密钥，而非范围独有密钥或文件独有密钥**，可为用户数据高效重建文件保险箱安全模型。用户仍须选择使用文件保险箱，以获得加密密钥层级搭配用户密码的全面保护。开发者也可以选择使用文件独有密钥或范围独有密钥的更高保护类。

> APFS 支持文件克隆（使用写入时拷贝技术的零损耗拷贝），如果文件被克隆，克隆的每一半都会得到一个新的密钥以接受传入的数据写入，这样新数据会使用新密钥写入媒介。久而久之， 文件可能会由不同的范围（或片段）组成，每个映射到不同的密钥（称为范围密钥，`Per-extend Key`）。
但是，组成文件的所有范围密钥仍然受同一 Class key 的保护。

搭载 Apple 芯片的 Mac 上的文件保险箱通过使用宗卷密钥的数据保护 C 类来实施。
在搭载 Apple T2 安 全芯片的 Mac 和搭载 Apple 芯片的 Mac 上， 直接连接到安全隔区的加密内部储存设备会使用安全隔区的硬件安全性功能和 AES 引擎的功能。
用户在 Mac 上启用文件保险箱后，启动过程中将需要其凭证。

### 1. 文件保险箱已打开的内部储存设备

如果没有有效的登录凭证或加密恢复密钥，即使物理储存设备被移除并连接到其他电脑，内置 APFS 宗卷仍保持加密状态，以防止未经授权的访问。在 macOS 10.15 中，此类宗卷同时包括系统宗卷和数据宗卷。从 macOS 11 开始，系统宗卷通过签名系统宗卷 (SSV) 功能进行保护，而数据宗卷仍通过加密进行保护。搭载 Apple 芯片的 Mac 以及搭载 T2 芯片的 Mac 通过构建和管理密钥层级实施内部宗卷加密，基于芯片内建的硬件加密技术而构建。
![filevault-on](filevault-on.png)

此密钥层级的设计旨在同时实现四个目标 :
• 请求用户密码用于加密
• 保护系统免受针对从 Mac 上移除的储存媒介的直接暴力破解攻击
• 提供擦除内容的快速安全的方法， 即通过删除必要的加密材料
• 让用户无需重新加密整个宗卷即可更改其密码 (同时也会更改用于保护其文件的加密密钥)

在搭载 Apple 芯片的 Mac 以及搭载 T2 芯片的 Mac 上，所有文件保险箱密钥的处理都发生在安全隔区中；加密密钥绝不会直接透露给 Intel CPU。 所有 APFS 宗卷默认使用宗卷加密密钥创建。宗卷和元数据内容使用此宗卷加密密钥加密，此宗卷加密密钥使用**类密钥**封装。 文件保险箱启用时，类密钥受用户密码和硬件 UID 共同保护。

### 2. 文件保险箱已关闭的内部储存设备

在搭载 Apple 芯片或搭载 T2 芯片的 Mac 上，如果在 “设置助理” 初次运行过程中未启用文件保险箱，宗卷仍会加密，但宗卷加密密钥仅由安全隔区中的硬件 UID 保护。
![filevault-off](filevault-off.png)

如果稍后启用了文件保险箱 (由于数据已加密， 该过程可立即完成)， 反重放机制会帮助阻止旧密钥 (仅基于硬件 UID) 被用于解密宗卷。 然后宗卷将受用户密码和硬件 UID 共同保护 (如前文所述)。

### 3. 删除文件保险箱宗卷

删除宗卷时，其宗卷加密密钥由安全隔区安全删除。这有助于防止以后使用此密钥进行访问 (即使是通过安全 隔区)。另外，所有宗卷加密密钥都使用媒介密钥封装。媒介密钥不提供额外的数据机密性，而是旨在启用快 速安全的数据删除，如果缺少了它，则不可能进行解密。
在搭载 Apple 芯片的 Mac 和搭载 T2 芯片的 Mac 上，媒介密钥一定是由受安全隔区支持的技术来抹掉，例如远程 MDM 命令。以这种方式抹掉媒介密钥将导致宗卷因存在加密而不可访问。

### 4. 可移除储存设备

可移除储存设备的加密不使用安全隔区的安全性功能，而是采用与基于 Intel 的 Mac (不搭载 T2 芯片) 相同的方式执行加密。

## 六、技术分析

### 1. SKP 密封密钥保护技术

Apple 设备支持一项称为密封密钥保护 (SKP，Sealed Key Prtection) 的技术， 其旨在确保加密材料在这些情况下不可用 : 脱离设备 时， 或者对操作系统版本或安全性设置存在未经用户正确授权的操纵时。

注意！SKP 功能不是由安全隔区提供，而是由位于更底层的硬件寄存器支持（也就意味着，仅在搭载 Apple 设计的 SoC 的设备上提供），目的是针对解密用户数据所需的密钥提供独立于安全隔区的额外保护层。

![SKP](SKP.png)

安全隔区启动监视器会捕获所加载的安全隔区操作系统的测量值。 当应用程序处理器 Boot ROM 测量附于 LLB 的 Image4 清单时， 该清单也包含所有其他已加载的系统配对固件的测量值。 LocalPolicy 

- 安全隔区固件 sepOS ：由安全隔区启动监视器（硬件实现）获取测量值
- LLB Image4 manifest：包含所有与 macOS 配对的固件和 macOS 核心启动对象 (如启动内核集或签名系统宗卷 (SSV) 根哈希值)的测量值
- 本地安全策略 LocalPolicy 的测量值：包含已加载 macOS 的核心安全性配置。 还包含 nsih 字段， 它是 macOS Image4 清单的哈希值。

![搭载 Apple 芯片的 Mac 开机时的启动过程步骤](mac-boot.png)

如果攻击者能够意外地更改任何上述测量的固件、 软件或安全性配置组件， 则也会修改储存在硬件寄存器中的测量值。 测量值的修改会导致从加密硬件派生的系统测量根密钥 (SMRK) 派生出不同的值， 从而有效破坏密钥层 级的封章。 这将导致无法访问系统测量设备密钥 (SMDK)， 从而导致无法访问 KEK， 因此也无法访问数据。

但是， 系统在未受到攻击时， 必须容纳合法的软件更新， 这些更新会更改固件测量值和 LocalPolicy 中的 nsih 字段， 以指向新的 macOS 测量值。 在其他尝试整合固件测量值但没有已知真实来源的系统中， 用 户将被要求停用安全性， 更新固件后重新启用安全性， 以便捕获新的测量基线。 这大大增加了攻击者在软件 更新期间篡改固件的风险。 Image4 清单包含所需的所有测量值， 这对系统很有帮助。 正常启动期间如果测 量值匹配， 使用 SMRK 解密 SMDK 的硬件也可以将 SMDK 加密为所建议的将来的 SMRK。 通过指定软 件更新后预期的测量值， 硬件可以加密在当前操作系统中可访问的 SMDK， 以便在将来的操作系统中仍可 访问。 同样地， 当客户在 LocalPolicy 中合法更改其安全性设置时， 必须根据 LLB 下次重新启动时计算的 LocalPolicy 测量值， 将 SMDK 加密为将来的 SMRK。

> Image4 是经 ASN.1（抽象语法标记一）DER 编码的数据结构格式，用于描述 Apple 平台上有关安全启动链对象的信息。

### 2. 关于 iOS 的文件保护类型

对于早期的 iOS 系统，Class D 是用户数据文件的默认类型，其类密钥是 DKey。也就是说，尽管每个文件都有 per-file key，但是这些文件专有密钥的包裹密钥都是 DKey，而 DKey 并未将 Passcode 作为加密因子（仅由基于 UID 的 Key 0x835 提供保护），因此其加密强度难以抵抗暴力破解。

从 iOS 5 开始，Apple 决定将 Class C 作为默认类型，即在设备开机时，校验用户输入的 passcode，成功后即可自由访问（即使发生了锁屏事件，仍然可以访问数据文件）；但是如果设备关机，就必须在开机之后重新校验 passcode。

Class A 主要适用于日历、信息、邮件、照片、通讯录和健康数据等个人敏感信息，其特点是一旦锁屏事件发生（延迟10秒），将丢弃内存中的类密钥，结果是无法访问数据文件了；只有用户重新输入 passcode 解锁屏幕后，再重新计算类密钥，才可以继续提供数据文件访问。

Class B 的设计目标是解决远程下载等互联网服务的后台权限问题，方法是采用 ECC 非对称密码系统，代价是文件元数据需要额外存储一个临时公钥（类密钥就是静态私钥）

### 2. 关于 MacOS 的 Volume Key



MacOS 使用的 FileVault 技术与 iOS 使用的 Data Protection 存在一些差异，主要体现在：

1. 不支持 Class D 保护类型，也就是说，没有 `DKey` 密钥；
2. Class C 是默认类型，但并非每个文件都有不同的 per-file key，而是所有文件内容使用统一的卷宗密钥（`Volume Key`）;
3. 

### 4. 遗留问题

1. 用户修改passcode后，需要重新封装metadata中所有的per-file key，似乎计算量也不小啊！

## 名词解释

### Effaceable Storage，可擦除区域

可擦除存储器 NAND 存储器中一个用于储存加密密钥的专用区域，可被直接寻址和安全擦除。尽管当攻击者实际占有设备时，可擦除存储器无法提供保护，但其中存储的密钥可用作密钥层级的一部分，用于实现快速擦 除和前向安全性。

### LLB - Low Level Bootloader，底层引导载入程序

在具有两步启动架构的 Mac 电脑上，LLB 包含由 Boot ROM 调用的代码，该代码随后会载入 iBoot，成为安全启动链的一环。

### xART - eXtended Anti-Replay Technology，反重放技术

一组为具有反重放功能 (基于物理储存架构) 的安全隔区提供加密且经认证的永久储存区的服务。

### 3. 核心密钥类型

### UID - unique ID，唯一ID

UID 是一个 256 位的 AES 密钥，在设备制造过程中刻录在每个处理器上。
这种密钥无法由固件或软件读取，只能由处理器的硬件 AES 引擎使用。 若要获取实际密钥，攻击者必须对处理器的芯片发起极为复杂且代价高昂的物理攻击。 UID 与设备上的任何其他标识符均无关，包括但不限于 UDID。

#### PDK - Passcode-Derived Key，密码派生密钥

用户密码与长期 SKP 密钥和安全隔区的 UID 配合使用，由此派生加密密钥。

#### per-file key，文件独有密钥

数据保护用于在文件系统上加密文件的密钥。文件独有密钥使用类密钥封装，储存在文件的元数据 metadata 中。

#### filesystem key，文件系统密钥

用于加密每个文件的元数据的密钥，包括其类密钥。存储在可擦除存储器中，用于实现快速擦除，并非用于保密目的。

#### Volume Key，卷宗密钥

数据宗卷文件系统中所有文件的元数据都使用随机宗卷密钥（Volume Key）进行加密，该密钥在首次安装操作系统或用户擦除设备时创建。

#### KEK - Key Encryption Key，密钥封装密钥

宗卷密钥密钥由密钥封装密钥（Key Encryption Key）加密和封装，密钥封装密钥由安全隔区长期储存，只在安全隔区中可见。每次用户抹掉设备时，它都会发生变化。在 A9（及后续型号） SoC 上，安全隔区依靠由反重放系统支持的熵来实现可擦除性，以及保护其他资源中的密钥封装密钥。
> 从功能描述上看，KEK 就是 PDK ！！！

---

## Apple官方文档

- [Apple 平台安全白皮书 - 2022年英文版](apple-platform-security-guide.pdf)
- [Apple 平台安全白皮书 - 2021年中文版](apple平台安全白皮书-2021中文版.pdf)
- [iOS 安全白皮书 - 2018年英文版](iOS安全白皮书-2018英文版.pdf)
- [Apple T2 安全芯片概览](Apple_T2_Security_Chip_Overview.pdf)
- [APFS 文件系统参考手册](Apple-File-System-Reference.pdf)
