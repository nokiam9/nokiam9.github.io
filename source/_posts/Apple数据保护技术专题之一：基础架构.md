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

钥匙包是一种用于储存一组类密钥的数据结构，有以下5种类型，但数据格式完全相同，包括：

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

钥匙串项使用两种不同的 AES-256-GCM 密钥加密 : 表格密钥 (元数据) 和行独有密钥 (私密密钥)。 钥匙串元数据 (除 kSecValue 外的所有属性) 使用元数据密钥加密以加速搜索，私密值 (kSecValueData) 使 用私密密钥进行加密。元数据密钥受安全隔区保护，但会缓存在应用程序处理器中以便进行钥匙串快速查询。私密密钥则始终需要通过安全隔区进行往返处理。
钥匙串以储存在文件系统中的`SQLite`数据库的形式实现， 而且数据库只有一个`securityd`监控程序决定每个进程或 App 可以访问哪些钥匙串项。钥匙串访问 API 将生成对监控程序的调用，从而查询 App 的 “keychain-access-groups”、 “application-identifier” 和 “application-group” 权限。访问组允许在 App 之间共享钥匙串项，而非将访问权限限制于单个进程。

### 钥匙串类型的定义

与文件保护类型对应的类型：

- kSecAttrAccessibleWhenUnlocked：未锁定状态下，对应文件保护的 Class A
- kSecAttrAccessibleAfterFirstUnlock：首次解锁后，对应文件保护的 Class C
- kSecAttrAccessibleAlways：始终，对应文件保护的 Class D

> 文件保护的 Class B 用于后台数据文件处理，此时没有前台业务，因此钥匙串没有对应的类型

与特定设备相关的类型：

其他钥匙串类都有对应的 “仅限本设备” 项目，后者在备份期间从设备拷贝时始终通过 UID 加以保护， 因此如果恢复至其他设备将无法使用。 
Apple 根据所保护信息的类型以及 iOS 和 iPadOS 需要这些信息的时间来选择钥匙串类，妥善平衡了安全性和可用性。
例如，VPN 证书必须始终可用，这样设备才能保持连接，但它归类为 “不可迁移”， 因此不能将其移至另一台设备

- kSecAttrAccessibleWhenUnlockedThisDeviceOnly：密码启用状态下
- kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly：
- kSecAttrAccessibleAlwaysThisDeviceOnly：

#### Keychain, Protection for build-in applications items

Item                                Accessibility

Wi-Fi passwords,                    Always
IMAP/POP/SMTP accounts,             AfterFirstUnlock
Exchange accounts,                  Always
VPN,                                Always
LDAP/CalDAV/CardDAV accounts,       lways
iTunes backup password,             WhenUnlockedThisDeviceOnly
Device certificate & private key,   AlwaysThisDeviceOnly

---

|可用性|文件数据保护|钥匙串数据保护|
|-|-|-|
||NSFileProtectionComplete|kSecAttrAccessibleWhenUnlocked|
|锁定状态下|NSFileProtectionCompleteUnlessOpen|暂无|
|首次解锁后|NSFileProtectionCompleteUntilFirstUserAuthentication|kSecAttrAccessibleAfterFirstUnlock|
|始终|NSFileProtectionNone|kSecAttrAccessibleAlways|
||暂无|kSecAttrAccessibleWhenPasscodeSet ThisDeviceOnly|

> 使用后台刷新服务的 App 可将 kSecAttrAccessibleAfterFirstUnlock 用于后台更新过程中需要访问的
钥匙串项。

iOS Keychain的分级保护机制与文件保护基本相同，但增加了 `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly`，该级别的`Class Key`在每次`Passcode`修改之后都会被清除，并且数据不会被备份。

- 类 kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly 与 kSecAttrAccessibleWhenUnlocked
行为方式相同；但前者仅当设备配置了密码时可用。 此类仅存在于系统密钥包中， 且它们：
• 不同步到 iCloud 钥匙串
• 不会备份
• 不包括在托管密钥包中
如果密码被移除或重设，类密钥便会丢弃，这些项目也变得无法使用。

- 


## 五、技术分析

1. 对于早期的 iOS 系统，Class D 是用户数据文件的默认类型，其类密钥是 DKey。也就是说，尽管每个文件都有 per-file key，但是这些文件专有密钥的包裹密钥都是 DKey，而 DKey 并未将 Passcode 作为加密因子（仅由基于 UID 的 Key 0x835 提供保护），因此其加密强度难以抵抗暴力破解。
2. 从 iOS 5 开始，Apple 决定将 Class C 作为默认类型，即在设备开机时，校验用户输入的 passcode，成功后即可自由访问（即使发生了锁屏事件，仍然可以访问数据文件）；但是如果设备关机，就必须在开机之后重新校验 passcode。
3. Class A 主要适用于日历、信息、邮件、照片、通讯录和健康数据等个人敏感信息，其特点是一旦锁屏事件发生（延迟10秒），将丢弃内存中的类密钥，结果是无法访问数据文件了；只有用户重新输入 passcode 解锁屏幕后，再重新计算类密钥，才可以继续提供数据文件访问。
4. Class B 的设计目标是解决远程下载等互联网服务的后台权限问题，方法是采用 ECC 非对称密码系统，代价是文件元数据需要额外存储一个临时公钥（类密钥就是静态私钥）

### 遗留问题

- 用户修改passcode后，需要重新封装metadata中所有的per-file key，似乎计算量也不小啊！


## 五、补充说明

- 数据保护通过构建和管理密钥层级来实施，并建立在 Apple 设备内建的硬件加密技术基础上。它通过将某个类分配给每个文件来实现对文件的逐个控制 ；可访问性根据该类密钥是否已解锁确定。APFS (Apple 文件系 统) 使文件系统可进一步以各个范围为基础对密钥进行细分 (文件的各个部分可以拥有不同的密钥)，称为`Per-extend Key`。
- 使用 APFS 格式运行的设备可能支持文件克隆（使用写入时拷贝技术的零损耗拷贝）。如果文件被克隆，克隆的每一半都会得到一个新的密钥以接受传入的数据写入，这样新数据会使用新密钥写入媒介。久而久之， 文件可能会由不同的范围（或片段）组成，每个映射到不同的密钥。但是，组成文件的所有范围受同一类密钥保护
- 数据宗卷文件系统中所有文件的元数据都使用随机宗卷密钥（`Volume Key`）进行加密，该密钥在首次安装操作系统或用户擦除设备时创建。此密钥由密钥封装密钥（`Key Wrapping Key`）加密和封装，密钥封装密钥由安全隔区长期储存，只在安全隔区中可见。每次用户抹掉设备时，它都会发生变化。 在 A9（及后续型号） SoC 上，安全隔区依靠由反重放系统支持的熵来实现可擦除性， 以及保护其他资源中的密钥封装密钥。
- 在搭载 Apple 芯片的 Mac 上，数据保护默认为 C 类， 但**使用宗卷密钥，而非范围独有密钥或文件独有密钥**，可为用户数据高效重建文件保险箱安全模型。用户仍须选择使用文件保险箱， 以获得加密密钥层级搭配用户密码的全面保护。开发者也可以选择使用文件独有密钥或范围独有密钥的更高保护类。

---

首先,iOS整个磁盘是全盘加密的，解密的EMF Key (file-system master encryption key)保存在闪存中1号扇区的可安全擦除区域(擦除后无法恢复，并且支持远程擦除)，该key对每台设备都是唯一，并且会在数据擦除时销毁；其次，在磁盘之上的文件系统的内容也是加密的， 在每个文件创建的时候会随机生成针对每个文件的Per-File Key，通过这个Per-File Key对文件中的内容进行加密保护，并且苹果又为此设计了Class Key来对Per-File Key进行保护。

无数据保护(Class D)
没有指定数据保护，但这不意味着文件没有加密保护，对于没有设置数据保护的其他所有文件，iOS中用一个DKey(Device Key)进行保护。该Key设备唯一，并且保存在闪存的可安全擦除区域。

锁屏密码(Passcode)的作用
假设把每个文件的加密看作为上了一道锁的话，那么对应的开锁的钥匙就存放在系统密钥包里面，而锁屏密码除了防止用户进入系统桌面之外，更重要的角色就是利用密码对系统密钥包进行额外的加密保护。很多人对锁屏密码理解的一个误区，就是锁屏密码只是物理手段上防止进入手机的一个保护，但实际上，用户在第一次设置锁屏密码的时候，锁屏密码会结合硬件加密引擎生成一个叫做Passcode Key的密钥，通过这个密钥对保存在密钥包中的各个钥匙(Class Key)进行加密保护。锁屏密码不会以其他加密的形式保存在设备上，用户在解锁的时候，会直接用输入的密码生成Passcode Key对密钥包中的Class Key解密，解密失败代表用户密码错误。

从苹果的数据加密和锁屏密码的保护机制来看，直接拆除存储芯片并对其进行文件读写操作是不可能的。

破解Passcode Key的手段
Passcode Key是用户输入的passcode结合系统硬件的加密引擎以及PBKDF2(Password-Based Key Derivation Function)算法生成的。PBKDF2 的基本原理是通过一个伪随机函数，把明文和一个盐值及加密重复次数作为输入参数，然后重复进行运算，并最终产生密钥。重复运算的会使得暴力破解的成本变得很高，而硬件key及盐值的添加基本上断绝了通过“彩虹表”攻击的可能 。

由于硬件加密引擎的Key无法提取，所以 只能在目标的机器上运行暴力破解程序进行破解，假设用户的密码设置的足够复杂的话，那么破解的周期就会变得非常久。

在FBI这个案例中，由于嫌犯可能开启了输错10次密码自动擦除设备的选项，一旦暴力猜测程序连续10次输入错误的密码，设备上的所有内容就会擦除掉。一旦触发数据擦除，苹果会首先对可安全擦除区域进行擦除，物理方式上即使能够恢复大部分加密数据，但是却无法恢复可安全擦除区域中的数据，因为大部分的解密密钥都保存在这个区域中，例如能够解开系统密钥包的二进制数据的BAG1 Key。

后续苹果为了封堵各种暴力猜测Passcode的方法，在64位设备的Secure Enclave中增加了定时器，针对尝试密码的错误次数，增加尝试的延时，即使断电重启也无法解决。

---


---

## 参考文献

### 资料下载




- [iOS 安全设计解析（二）：数据加密和保护](https://blog.blupig.net/ios-security-encryption)
- [iOS资料保护机制简介](https://www.kaotenforensic.com/ios/ios-data-protection/)
- [苹果的锁屏密码到底有多难破？ - 头条](https://toutiao.io/posts/324007/app_preview)
