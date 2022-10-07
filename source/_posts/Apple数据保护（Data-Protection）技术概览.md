---
title: Apple数据保护（Data Protection）技术概览
date: 2022-10-05 21:19:00
tags:
---

安全启动链、系统安全性和App安全性功能有助于验证只有受信任的代码及App可以在设备上运行，但Apple还需要更多加密功能实现用户数据的安全保护，主要挑战包括：

1. iPhone等移动终端设备保存通讯录、照片等大量个人信息，与Mac相比，对数据安全保护的要求更高、更细致；
2. App Store、iCloud、在线软件升级等云服务大量出现，网络开放环境下如何平衡数据安全和服务便利的矛盾；
3. 跨平台的生态系统需要密切协同，多个不同设备之间如何解决服务协同与数据安全的矛盾；

作为数据保护的基础，1998年Apple开始使用的 HFS+（MacOS扩展日志式）被称为最烂的文件系统，设计源自软盘和磁盘设备时代，甚至不支持文件名大小写敏感。
2014年，在 Giampaolo 的带领下，Apple启动了APFS（Apple File System）项目研发，并于2017年首次发布，短板终于修补上了！
APFS号称是针对SSD等新型设备专门设计（也兼容传统硬盘），提供Copy-on-Write、系统快照、动态分区调整、稀疏大文件支持等功能，但与业界最优秀的ZFS相比并无优势，唯一值得表扬的是，结合Apple封闭硬件体系的优势，数据加密功能大大提升，iPhone再也不怕被执法机构读取数据了。

关于数据安全，尤其是智能终端设备，Apple的主要设计目标包括：

- 在硬件被改动的情况下也能保护数据（替换组件/直接读取闪存等）
- 对抗离线攻击（物理方式获取闪存内的数据后用更强大的计算设备进行破解）
- 对不同的数据提供不同级别的保护（锁屏之后有些数据要保护，有些数据还需要访问）
- 需要时能够快速安全清除所有数据
- 考虑性能和功耗

目前，iOS 和iPadOS设备使用称为数据保护（Data Protection）的文件加密方法，基于Intel的Mac设备通过文件保险箱（FileVault）的宗卷加密技术进行数据保护，搭载Apple芯片的Mac设备使用两者的混合模型，但具体功能上有所差异。
此外，Apple还使用操作系统内核强制执行保护和安全措施。内核使用访问控制来沙盒化App(限制App可访问的数据)，还使用一种称为数据保险箱的机制来限制所有其他提出请求的App访问某一App 的数据(而不是限制某个App可进行的调用)。

## 一、文件数据保护

iOS 的所有用户数据文件都是加密存储的，Mac的情况复杂一些，文件系统加密是可选的，但USB外接存储就没有必要了。主要设计思想是：

- 每个文件创建时会生成一个`Per-file Key`，在文件写入闪存时通过硬件用`AES-XTS`加密
- `Per-file Key`存储在文件的元数据 (Metadata) 中，被`Class Key`和`Filesystem Key`加密
- `Class Key`和`Filesystem Key`都被硬件密钥`UID`保护
- `Class Key`还被用户密码`Passcode`保护

> When a file is opened, its metadata is decrypted with the file system key, revealing the wrapped per-file key and a notation on which class protects it. The per-file (or per-extent) key is unwrapped with the class key and then supplied to the hardware AES Engine, which decrypts the file as it’s read from flash storage. All wrapped file key handling occurs in the Secure Enclave; the file key is never directly exposed to the Application Processor.

![key](key.png)

这种设计方案的主要优势在于：

1. 实现了“一件一密”，每个文件都有独立的密钥`Per-file Key`，加密强度大大提高
2. `Class Key`实现对文件不同级别的保护（最高级别只有在设备解锁状态下才可用），一个密钥就可以解封对应级别所有文件的`Per-file Key`；当用户修改`Passcode`时，只需重置`Class Key`并对元数据中的文件密钥重新封装即可，原始文件无需重新加密，兼顾了安全性和灵活性
3. `Filesystem Key`在 iOS 安装时或每次设备数据被清除时生成，其目标不是保护数据的机密性，而是用于实现快速清除数据，即一旦`Filesystem Key`被删除，设备上所有文件将永久无法解密
4. 所有加密处理由 AES Engine 的硬件处理，性能有保障；同时所有密钥由 Secure Enclave 统一管理，不对外泄露，确保安全性并对上层应用透明
5. 文件数据有`Class Key`和`Filesystem Key`两层密钥，并最终由`UID`和`Passcode`提供保护，考虑到用户设置`Passcode`的强度不高，Apple还专门强化了安全隔区的反重放机制以提高安全性

> 有点疑问：用户修改passcode后，是否需要重新封装metadata中所有的per-file key？好像计算量也不小啊！！！

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

- 此类密钥仅受 UID 的保护，并且存储在可擦除存储器中。由于解密该类中的文件所需的所有密钥都储存在设备上，因此采用该类加密的唯一好处就是可以进行快速远程擦除。即使未向文件分配数据保护类，此文件仍会以加密形式储存 （就像 iOS 和 iPadOS 设备上的所有数据那样）。
- macOS 不支持该类型。

小结一下：

- C类是新建文件的默认类型，即用户开机后可以自由访问，重点是防止设备遗失后被第三方读取数据，同时对软件开发也没有影响
- B类的设计目标是解决远程下载等互联网服务的后台权限问题，方法是采用公钥+私钥的非对称加密算法
- A类的技术代价高，估计使用范围有限，主要适用于日历、信息、邮件、照片、通讯录和健康数据等个人敏感信息
- D类就是不加密了，比较适合系统备份，或USB外接存储

> The contents of a file may be encrypted with one or more per-file (or per-extent) keys that are wrapped with a class key and stored in a file‘s metadata, which in turn is encrypted with the file system key. The class key is protected with the hardware UID and, for some classes, the user’s passcode. This hierarchy provides both flexibility and performance. For example, **changing a file‘s class only requires rewrapping its per-file key, and a change of passcode just rewraps the class key**。

## 三、钥匙包（keyBag）数据保护

钥匙包是一种用于储存一组类密钥的数据结构，有以下5种类型，但数据格式完全相同，包括：

- 标头：版本号，钥匙包类型，密钥包 UUID，HMAC （若密钥包已签名），用于封装类密钥的方法 ：配合盐和迭代计数使用 Tangling 及UID 或 PBKDF2。
- 类密钥列表 ：密钥 UUID ；类 （哪个文件或钥匙串数据保护类）；封装类型 （仅 UID 派生密钥 ； UID 派生密钥和密码派生密钥）；封装的类密钥 ；非对称类的公钥。

• A header containing: 
    – Version (set to three in iOS 5) 
    – Type (system, backup, escrow, or iCloud Backup) 
    – Keybag UUID 
    – An HMAC if the keybag is signed 
    – The method used for wrapping the class keys—tangling with the UID or PBKDF2,
    along with the salt and iteration count

• A list of class keys: 
– Key UUID 
– Class (which file or Keychain Data Protection class this is) 
– Wrapping type (UID-derived key only; UID-derived key and passcode-derived key) 
– Wrapped class key 
– Public key for asymmetric classes

### 1. 用户密钥包(User keyBag)

用户密钥包是设备常规操作中使用的封装类密钥的储存位置。例如，输入密码后，`NSFileProtectionComplete`会从用户密钥包中载入并解封。它是储存在“无保护”类中的二进制属性列表 (.plist) 文件。
对于搭载 A9 之前的 SoC 的设备，该 .plist 文件的内容通过保存在可擦除存储器中的密钥加密。为了给密钥包提供前向安全性，用户每次更改密码时，系统都会擦除并重新生成此密钥。
对于搭载 A9 或后续型号 SoC 的设备，该 .plist 文件包含一个密钥，表示密钥包储存在受反重放随机数（由安全隔区控制）保护的有锁储存库中。

### 2. 设备密钥包 (Device keyBag)

设备密钥包用来储存用于涉及设备特定操作数据的封装类密钥。 配置为共用的 iPadOS 设备有时需要在用户登录前访问凭证，因此需要一个不受用户密码保护的密钥包。
iOS 和 iPadOS 不支持对用户独有的文件系统内容进行单独加密，这就意味着系统使用来自设备密钥包的类密钥，对文件独有密钥进行封装，而钥匙串则使用来自用户密钥包中的类密钥来保护用户钥匙串中的项目。
在配置为单用户使用 (默认配置) 的 iOS 和 iPadOS 设备中， 设备密钥包和用户密钥包是同一个， 并受用户的密码保护。

### 3. 备份密钥包（Backup keyBag:）

备份密钥包在 “访达” (macOS 10.15 或更高版本) 或 iTunes (macOS 10.14 或更低版本) 进行加密备份时创建，并储存在设备被备份到的电脑中。
新密钥包是通过一组新密钥创建的，备份的数据会使用这些新密钥重新加密。 如前所述，不可迁移钥匙串项仍使用 UID 派生密钥封装，以使其可以恢复到最初备份它们的设备，但在其他设备上不可访问。
由于备份密钥包并未捆绑特定设备， 理论上尝试在多台电脑上对备份密钥包并行展开暴力破解是可行的。 
如果用户选择不加密备份， 那么不管备份文件属于哪一种数据保护类， 备份文件都不加密，但钥匙串仍使用 UID 派生密钥获得保护。这就是只有设置备份密码才能将钥匙串项迁移到新设备的原因。

### 4. 托管密钥包（Escrow keybag）

托管密钥包用于通过 USB 与 “访达” (macOS 10.15 或更高版本) 或 iTunes (macOS 10.14 或更低版 本) 进行同步， 还用于移动设备管理 (MDM)。 此密钥包允许 “访达” 或 iTunes 执行备份和同步， 而无需用户 输入密码， 它还允许 MDM 解决方案远程清除用户密码。 它储存在用于与 “访达” 或 iTunes 进行同步的电脑 上， 或者在远程管理设备的 MDM 解决方案上。
托管密钥包改善了设备同步过程中的用户体验， 期间可能需要访问所有类别的数据。 当使用密码锁定的设备首 次连接到 “访达” 或 iTunes 时， 会提示用户输入密码。 然后设备创建托管密钥包， 其中包含的类密钥与设备 上使用的完全相同， 该密钥包由新生成的密钥进行保护。 托管密钥包及用于保护它的密钥划分到设备和主机 或服务器上， 其数据以 “首次用户认证前保护” 类储存在设备上。 这就是重新启动后， 用户首次使用 “访达” 或 iTunes 进行备份之前必须输入设备密码的原因。

### 5.云备份密钥包（iCloud Backup keybag）

iCloud 云备份密钥包与备份密钥包类似。 该密钥包中的所有类密钥都是非对称的 (与 “未打开文件的保护”数据保护类一样， 使用 Curve25519)。 
非对称密钥包还可用于 “iCloud 钥匙串” 钥匙串恢复中的备份。

## 四、钥匙串 (Key Chain)数据保护

许多 App 都需要处理密码和其他一些简短但比较敏感的数据，如密钥和登录令牌。钥匙串提供了储存这些项 的安全方式。不同的 Apple 操作系统采用不同机制实施与各钥匙串保护类关联的保障。
在 macOS (包括搭 载 Apple 芯片的 Mac) 中，数据保护不直接用于实施此类保障。

钥匙串项使用两种不同的 AES-256-GCM 密钥加密 : 表格密钥 (元数据) 和行独有密钥 (私密密钥)。 钥匙串元数据 (除 kSecValue 外的所有属性) 使用元数据密钥加密以加速搜索，私密值 (kSecValueData) 使 用私密密钥进行加密。元数据密钥受安全隔区保护，但会缓存在应用程序处理器中以便进行钥匙串快速查询。私密密钥则始终需要通过安全隔区进行往返处理。
钥匙串以储存在文件系统中的`SQLite`数据库的形式实现， 而且数据库只有一个,`securityd`监控程序决 定每个进程或 App 可以访问哪些钥匙串项。钥匙串访问 API 将生成对监控程序的调用，从而查询 App 的 “keychain-access-groups”、 “application-identifier” 和 “application-group” 权限。访问组允许在 App 之间共享钥匙串项，而非将访问权限限制于单个进程。

|可用性|文件数据保护|钥匙串数据保护|
|-|-|-|
|未锁定状态下|NSFileProtectionComplete|kSecAttrAccessibleWhenUnlocked|
|锁定状态下|NSFileProtectionCompleteUnlessOpen|暂无|
|首次解锁后|NSFileProtectionCompleteUntilFirstUserAuthentication|kSecAttrAccessibleAfterFirstUnlock|
|始终|NSFileProtectionNone|kSecAttrAccessibleAlways|
|密码启用状态下|暂无|kSecAttrAccessibleWhenPasscodeSet ThisDeviceOnly|

iOS Keychain的分级保护机制与文件保护基本相同，但增加了 `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly`，该级别的`Class Key`在每次`Passcode`修改之后都会被清除，并且数据不会被备份。

## 五、补充说明

- 数据保护通过构建和管理密钥层级来实施，并建立在 Apple 设备内建的硬件加密技术基础上。它通过将某个类分配给每个文件来实现对文件的逐个控制 ；可访问性根据该类密钥是否已解锁确定。APFS (Apple 文件系 统) 使文件系统可进一步以各个范围为基础对密钥进行细分 (文件的各个部分可以拥有不同的密钥)，称为`Per-extend Key`。
- 使用 APFS 格式运行的设备可能支持文件克隆（使用写入时拷贝技术的零损耗拷贝）。如果文件被克隆，克隆的每一半都会得到一个新的密钥以接受传入的数据写入，这样新数据会使用新密钥写入媒介。久而久之， 文件可能会由不同的范围（或片段）组成，每个映射到不同的密钥。但是，组成文件的所有范围受同一类密钥保护
- 数据宗卷文件系统中所有文件的元数据都使用随机宗卷密钥（`Volume Key`）进行加密，该密钥在首次安装操作系统或用户擦除设备时创建。此密钥由密钥封装密钥（`Key Wrapping Key`）加密和封装，密钥封装密钥由安全隔区长期储存，只在安全隔区中可见。每次用户抹掉设备时，它都会发生变化。 在 A9（及后续型号） SoC 上，安全隔区依靠由反重放系统支持的熵来实现可擦除性， 以及保护其他资源中的密钥封装密钥。
- 在搭载 Apple 芯片的 Mac 上，数据保护默认为 C 类， 但**使用宗卷密钥，而非范围独有密钥或文件独有密钥**，可为用户数据高效重建文件保险箱安全模型。用户仍须选择使用文件保险箱， 以获得加密密钥层级搭配用户密码的全面保护。开发者也可以选择使用文件独有密钥或范围独有密钥的更高保护类。

---

## 参考文献

- [iOS 安全设计解析（二）：数据加密和保护](https://blog.blupig.net/ios-security-encryption)
- [iOS资料保护机制简介](https://www.kaotenforensic.com/ios/ios-data-protection/)
