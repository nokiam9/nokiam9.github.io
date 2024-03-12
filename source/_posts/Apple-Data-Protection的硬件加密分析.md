---
title: Apple Data Protection的硬件加密分析
date: 2024-03-04 21:23:05
tags:
---

根据 Apple 官网提供的[第三方平台认证信息](https://support.apple.com/zh-cn/guide/certifications/welcome/web)，其硬件、操作系统、应用软件、互联网服务和 Apple Pay 均符合 FIPS 140-2 认证，但 FIPS 140-3 认证仍在进行中，[安全隔区的加密模块报告](Apple_Secure_Key_Store_Cryptographic_Module_v10.0.pdf)的最新版本是 v10.0。
此外，Apple 也符合通用标准 (CC) 认证，并委托知名评估机构 [ATSEC 公司](https://www.atsec.com)出具了[iOS 16 安全评估报告 TOE - Target of Evaluation](https://www.niap-ccevs.org/MMO/Product/st_vid11349-st.pdf)，当前版本是 v1.1。
本文就是依据上述公开信息的研究分析。

---

|Key / Persistent Secret|Purpose|Storage (for all devices)|
|---|---|---|
|UID|REK for device Key，entanglement|SEP|
|Salt (128 bits) |Additional input to one- way functions|AES encrypted in the system keybag|
|Key 0x89B|Wrapping of EMF key Wrapping of Dkey|SEP. Block 0 of the flash memory. (Effaceable storage.)|
|Key 0x835|Used for the encryption of file system metadata|SEP. Block 0 of the flash memory. (Effaceable storage.)|
|EMF key|Writing files while the device is locked|Stored in wrapped form in persistent storage|
|NSFileProtectionCompleteUnlessOpen devicewide asymmetric key pair|Writing files while the device is locked|Stored in wrapped form in persistent storage|
|CompleteUntilFirstUserAuthentication||Stored in wrapped form in persistent storage|
|NSFileProtectionCompleteUnlessOpen|Writing files while the device is locked: KDF static public keys|Stored in wrapped form in persistent storage|
|AfterFirstUnlock||Stored in wrapped form in persistent storage|
|AfterFirstUnlockThisDeviceOnly||Stored in wrapped form in persistent storage|
|WhenUnlocked||Stored in wrapped form in persistent storage|

---

### 保护类型对应关系

|Keychain的数据保护类型|File的数据保护类型|适用场景|
|:---:|:---:|:---:|
|kSecAttrAccessibleWhenUnlocked|NSFileProtectionComplete|未锁定状态|
|N/A |NSFileProtectionCompleteUnlessOpen|锁定状态|
|kSecAttrAccessibleAfterFirstUnlock|NSFileProtectionCompleteUntilFirstUserAuthentication |首次解锁后|
|kSecAttrAccessibleAlways|NSFileProtectionNone|始终|

- 使用后台刷新服务的 App 可将`kSecAttrAccessibleAfterFirstUnlock`用于后台更新过程中需要访问的钥匙串项。
- 上述三钟钥匙串类都有对应的`ThisDeviceOnly`项目，后者在备份期间从设备拷贝时始终通过 UID 加以保护，因此如果恢复至其他设备将无法使用，例如 VPN 证书不适合迁移至另一台设备。
- `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly`与`kSecAttrAccessibleWhenUnlocked`的行为方式相同，但前者仅当设备配置了密码时可用。此类仅存在于系统密钥包中， 且它们不同步到 iCloud 钥匙串、不会备份、不包括在托管密钥包中。

### asd

- 本节中讨论的密钥由 SEP 管理和维护。OS 和 SEP 使用 mailbox 进行数据交互
- 所有的文件系统 item 和所有的钥匙串 item 仅以加密形式存储
- 使用 EMF 密钥对文件系统 meatdata 进行加密
- 文件和钥匙串的 item 使用独立密钥进行加密。这些键与项所属的类、文件或钥匙串的类键一起包装。
- 属于`NSFileProtectionNone`类的文件，和属于`Always`或`AlwaysThisDeviceOnly`类的钥匙串项，仅使用 `Dkey`包裹的密钥进行加密。在对用户进行身份验证之前，可以访问（解密）这些项目
  对于所有其他类，`passcode key`用于生成用于这些类的包裹密钥，因此，只有当用户正确输入 passcode 时，才能解密这些项目
- 所有解密错误均按照 [SP800-56B-Rev1] 进行处理
- 发出擦除命令时，将通过擦除**顶级 KEK** 来擦除受保护的数据。由于所有静态数据都使用其中一个密钥进行加密，因此会擦除设备

>- The keys discussed in this section are managed by and/or maintained in the SEP. The TOE OS and SEP interact with each other using a mailbox system detailed in Section 7.1.1.1.
>- All file system items and all keychain items are stored in encrypted form only.
>- File system metadata is encrypted using the EMF key.
>- Files and keychain items are encrypted with individual keys. Those keys are wrapped with the class key of the class, the file, or the Keychain to which the item belongs.
>- Files and keychain items belonging to the classes 'NSFileProtectionNone' (files) and 'Always' or 'AlwaysThisDeviceOnly' are encrypted with keys that are wrapped with the Dkey only. Those items can be accessed (decrypted) before the user is authenticated. For all other classes, the passcode key (which is derived from the user's passcode) is used in the generation of the wrapping key used for those classes, and therefore, decrypting those items is only possible when the user has correctly entered their passphrase.
>- All decryption errors are handled in compliance with [SP800-56B-Rev1].
>- When a wipe command is issued, protected data is wiped by erasing the top-level KEKs. Since all data at rest is encrypted with one of those keys, the device is wiped.

### TOE 启动流程

1. 使用安全隔区的 TRNG 在 SEP 中创建一个 256 位的`临时封装密钥`
2. OS 内核从 Effaceable Stroage 读取**包裹状态**的 `Dkey` 和 `EMF key`，并发送至 SEP
3. SEP 持有`Key 0x835`解封`Dkey` ，持有`Key 0x89B`解封 `EMF key`
4. SEP 持有 Step 1 生成的`临时封装密钥`包裹`Dkey` #TODO:
5. SEP 通过专用线路（I2C协议）将`临时封装密钥`传输到 AES 引擎。OS 内核无法访问此区域。

>- An ephemeral AES key (256-bit) is created in the SEP using the random bit generator of the Secure Enclave.
>- The (wrapped) Dkey and (wrapped) EMF key (both 256-bit keys) are loaded by the TOE OS kernel from the effaceable storage and sent to the SEP.
>- The SEP unwraps the Dkey and the EMF key.
>- The SEP wraps the Dkey with the newly generated ephemeral key.
>- The SEP stores the ephemeral key in the storage controller. This area is not accessible by the TOE OS kernel.

### TOE OS 读取文件流程

1. OS 内核首先提取**加密状态**的文件 meatdata 并将其发送到 SEP
2. SEP 持有`EMF Key`解密文件 meatdata，并将其发送回 OS 内核
3. OS 内核分析文件 meatdata 确定数据保护类别，并将**包裹状态**的`Class Key` 和`per-file key`一起发送到 SEP
    - Class D 的类密钥基于`Dkey`包裹，其他类密钥基于`Dkey XOR Passcode Key`包裹
    - `per-file key`基于指定的`Class Key`包裹
4. SEP 持有`Dkey`和`Passcode Key`解封`per-file key`，使用`临时封装密钥`重新包裹它，并发送回 OS 内核
5. OS 内核将文件访问请求（读取或写入）和**重新包裹状态**的`per-file key`一起发送到存储控制器
6. 存储控制器使用 AES 引擎持有的`临时封装密钥`解封`per-file key`，然后在数据从闪存传输/传入闪存期间解密（读取操作时）或加密（写入操作时）

>- The TOE OS kernel first extracts the file metadata (which are encrypted with the EMF key) and sends them to the SEP.
>- The SEP decrypts the file metadata and sends it back to the TOE OS kernel.
>- The TOE OS kernel determines which class key to use and sends the class key (which is wrapped with the Dkey, or with the XOR of the Dkey and the Passcode Key) and the file key (which is wrapped with the class key) to the SEP.
>- The SEP unwraps the file key and re-wraps it with the ephemeral key and sends this wrapped key back to the TOE OS kernel.
>- The TOE OS kernel sends the file access request (read or write) together with the wrapped file key to the storage controller.
>- The storage controller uses its internal implementation of AES, decrypts the file key, and then decrypts (when the operation is read) or encrypts (when the operation is write) the data during its transfer from/to the flash memory.

### 密钥包

文件和钥匙串数据保护类的密钥均在密钥包中收集和管理。 TOE 操作系统使用以下密钥包：系统（也称为用户和设备密钥包）、备份、托管和 iCloudBackup。 密钥存储在系统密钥袋中，部分密钥存储在托管密钥袋中。 托管密钥包用于设备更新和 MDM，这都是 [MDF]☝ 中定义的相关功能。

系统密钥包是存储设备正常操作中使用的包装类密钥的位置。 例如，当输入密码或生物识别身份验证因素时，将从系统密钥包加载 NSFileProtectionComplete 密钥并解开包装。 它是存储在无保护类中的二进制 plist，但其内容使用可擦除存储中保存的密钥（`Dkey`）进行加密。 为了给密钥包提供前向安全性，每次用户更改密码时都会擦除并重新生成该密钥。

**AppleKeyStore 内核扩展管理系统密钥包**，并且可以查询设备的锁定状态。 它报告说，只有当系统密钥包中的所有类密钥都可以访问并且已成功解开包装时，设备才会解锁。

### 分析

- UID 存储在 SEP 固件中，位于 SEP 或应用处理器中任何程序都无法访问的部分。SEP 只能用于使用 UID 作为密钥来加密和解密数据（使用 AES-256）。
- `Key 0x89B`和`Key 0x835`存储在 SEP 中（估计是在 SEP 启动时一次性计算产生）
- `EMF Key`、`Dkey` 和**类密钥（？BAG1）**存储在 Effaceable Stroage 中，全部仅以包裹状态存储。如前所述，它们在应用处理器系统中永远不会以明文形式提供。
- 文件密钥和钥匙串项密钥存储在内部非易失性存储器（NAND）中，但仅以包裹状态存储。如前所述，它们在应用处理器系统中永远不会以明文形式提供。
- 系统和应用程序可以将私钥存储在钥匙串项中。它们受钥匙串项加密的保护。
- 用于 TLS、HTTPS 或 Wi-Fi 会话的对称密钥仅保存在 RAM 中。同样，用于 TLS 和 HTTPS 的 ECDH 非对称密钥仅保存在 RAM 中。它们使用两个库之一生成和管理，即`Apple corecrypto 模块 v13.0 [Apple ARM、用户、软件、SL1]`和`Apple corecrypto 模块 v13.0 [Apple ARM、内核、软件、SL1]`，或由 Wi-Fi 芯片中的 AES 实现。这些库的函数，比如 `memset（0）`，也会在使用后清除这些密钥。

>- The UID is stored in the firmware of the SEP in a section not accessible by any program in the SEP or the application processor. The SEP can only be used to encrypt and decrypt data (with AES-256) using the UID as the key.
>- "Key 0x89B" and "Key 0x835" are stored in the SEP.
>- The EMF key, Dkey, and the class keys are stored in the effaceable area, all in wrapped form only. As explained, they are never available in plaintext in the application processor system.
>- File keys and Keychain item keys are stored in internal, non-volatile memory, but in wrapped form only. As explained, they are never available in plaintext in the application processor system.
>- The system and the applications can store private keys in Keychain items. They are protected by the encryption of the Keychain item.
>- Symmetric keys used for TLS, HTTPS, or Wi-Fi sessions are held in RAM only. Similarly, ECDH asymmetric keys used for TLS and HTTPS are held in RAM only. They are generated and managed using one of the two libraries, Apple corecrypto Module v13.0 [Apple ARM, User, Software, SL1] and Apple corecrypto Module v13.0 [Apple ARM, Kernel, Software, SL1], or by the AES implementation within the Wi-Fi chip. The functions of those libraries, such as memset(0), also perform the clearing of those keys after use.

## 一、整体架构

Cocoa 是苹果公司为 macOS 所创建的原生面向对象的应用程序接口，是 Mac OS X 上五大 AP 之一（其它四个是Carbon、POSIX、X11和Java）。

Cocoa 应用程序一般在苹果公司的开发工具 Xcode（前身为 Project Builder ）和 Interface Builder 上用Objective-C 写成。

Cocoa包含三个主要的Objective-C对象库，称为“框架”。框架的功能类似于动态库，即可以在运行时动态的加载应用程序的地址空间，但框架作为一个捆绑而非独立文件，其中除了可执行代码外，也包含了资源，头文件和文档。

“Foundation工具包”，或简称为“Foundation”，首先出现在OpenStep中。在Mac OS X中，它是基于Core Foundation的。作为通用的面向对象的函数库，Foundation提供了字符串，数值的管理，容器及其枚举，分布式计算，事件循环，以及一些其它的与图形用户界面没有直接关系的功能。其中用于类和常数的“NS”前缀来自于Cocoa的来源，NeXTSTEP。它可以在Mac OS X和iOS中使用。
“应用程序工具包”，或称AppKit（Application Kit）是直接派生自NeXTSTEP的AppKit的。它包含了程序与图形用户界面交互所需的代码。它是基于Foundation创建的，也使用“NS”前缀。它只能在Mac OS X中使用。
“用户界面工具包”，或称UIKit（User Interface Kit），是用于iOS的图形用户界面工具包。与AppKit不同，它使用“UI”的前缀。

根据[iOS 16 安全评估报告](Apple_iOS_16_iPhone_Security_Target.pdf)的表述，其提供三个层次的加密服务：

- User Space：SL1，软件级。驻留在用户空间内的动态可加载库，为 App 提供加密功能
- Kernel Space：SL1，软件级。操作系统内核加载的扩展模块，仅为内核提供加密功能
- Secure Key Store：SL2，硬件级，也称为 SKS（安全密钥存储）。一个单芯片独立硬件加密模块，为传输中和静态数据提供保护，包含固件和硬件加密算法实现

![SKS](SKS-arch.png)

- HW：hardware，硬件；
- POST：Power-On Self Test，启动自检；
- OTP-ROM：One Time Programmable Read-Only Memory，一次性编程的只读内存，即熔丝；
- DRBG，Deterministic Random Bit Generator，确定性随机位生成器，即伪随机发生器

在具有A12、S4和更高版本SoC的设备中，安全飞地与用于熵存储的安全存储组件配对。
安全存储组件本身设计有：不可变的ROM代码、硬件随机数生成器、每个设备唯一的加密密钥、加密引擎和物理篡改检测。

2020年秋季或以后首次发布的设备配备了第二代安全存储组件。
第二代安全存储组件增加了计数器锁箱。
每个计数器锁箱存储128位salt、128位密码验证器、8位计数器和8位最大尝试值。

---

![TOC](<TOC-key.png>)

### 安全隔区 AES 引擎

安全隔区 AES 引擎是一个硬件块， 用于基于 AES 密码来执行对称加密。 AES 引擎设计用于防范通过时序和静态功耗分析 (SPA) 泄露信息。 从 A9 SoC 开始， AES 引擎还包括动态功耗分析 (DPA) 对策。
AES 引擎支持硬件密钥和软件密钥。 **硬件密钥派生自安全隔区 UID 或 GID**。 这些密钥保存在 AES 引擎内，即使对 sepOS 软件也不可见。 虽然软件可以请求通过硬件密钥进行加密和解密操作， 但不能提取密钥。

### AES 引擎

每台包含安全隔区的 Apple 设备还具有专用的 AES256 加密引擎 （“AES 引擎”）， 它内置于 NAND（非易失性） 闪存与主系统内存之间的直接内存访问 (DMA) 路径中， 可以实现高效的文件加密。 在 A9或后续型号的 A 系列处理器上， 闪存子系统位于隔离的总线上， 该总线被授权只能通过 DMA 加密引擎访问包含用户数据的内存。

启动时， sepOS 会使用 TRNG 生成一个**临时封装密钥**。 安全隔区**使用专用线**将此密钥传输到 AES 引擎，旨在防止它被安全隔区外的任何软件访问。 sepOS 随后可以使用临时封装密钥来封装文件密钥， 供应用程序处理器文件系统驱动程序使用。 当文件系统驱动程序读取或写入文件时， 它会将封装的密钥发送到 AES 引擎以解封密钥。 AES 引擎绝不会将未封装的密钥透露给软件。

### 安全非易失性存储器 - Secure nonvolatile storage

安全隔区配备了专用的安全非易失性存储器设备。 安全非易失性存储器通过专用的 I2C 总线与安全隔区连接，因此它仅可被安全隔区访问。**所有用户数据加密密钥植根于储存在安全隔区非易失性存储器中的熵内**。

搭载 A12、 S4 及后续型号 SoC 的设备并用了安全隔区与安全储存组件来储存熵。 安全储存组件本身设计为使用：

- 不可更改的 ROM 代码
- 硬件随机数生成器
- **每个设备唯一的加密密钥**
- 加密引擎
- 物理篡改检测
- 计数器加密箱（第二代增加）：包含 salt、verify、counter、max_retry_limit

安全隔区和安全储存组件使用加密且认证的协议通信以提供对熵的独有访问权限。

2020 年秋季或之后首次发布的设备配备了**第二代安全储存组件**。 第二代安全储存组件增加了计数器加密箱。每个计数器加密箱储存一个 128 位**盐**、 一个 128 位**密码验证器**、 一个 8 位计数器， 以及一个 8 位最大尝试值。 对计数器加密箱的访问通过加密且认证的协议来实现。

计数器加密箱中含有所需用于解锁受密码保护用户数据的熵。 若要访问用户数据， 配对的安全隔区必须从用户的密码和安全隔区的 UID 中派生出正确的密码熵值。 从除配对安全隔区之外其他来源发送的解锁尝试均无法获知用户的密码。 如果密码的尝试次数超过限制 （例如， 在 iPhone 上为 10 次）， 安全储存组件就会完全抹掉受密码保护的数据。

为了创建计数器加密箱， 安全隔区会向安全储存组件发送密码熵值和最大尝试次数值。 安全储存组件会**使用其随机数生成器生成盐值**。 之后**通过提供的密码熵、 安全储存组件的唯一加密密钥和盐值派生出密码验证器值和加密箱熵值**。 安全储存组件使用计数 0、 提供的最大尝试次数值、 派生的密码验证器值和盐值来初始化计数器加密箱。 之后安全储存组件将生成的加密箱熵值返回到安全隔区。

为了稍后从计数器加密箱中取回加密箱熵值， 安全隔区会向安全储存组件发送密码熵。 安全储存组件会先为加密箱递增计数器。 如果递增后的计数器超过最大尝试次数值， 安全储存组件就会完全抹掉计数器加密箱。 如果尚未达到最大尝试次数， 安全储存组件会尝试通过与用于创建计数器加密箱相同的算法来派生出密码验证器值和加密箱熵值。 如果派生的密码验证器值匹配所储存的密码验证器值， 安全储存组件会将加密箱熵值返回到安全隔区并将计数器重设为 0。

用于访问受密码保护数据的密钥植根于计数器加密箱所储存的熵中。 有关更多信息， 请参阅数据保护概览。
安全隔区中的所有反重放服务均使用了安全非易失性存储器。 安全隔区上的反重放服务用于在发生以下标记反重放边界的事件时撤销数据， 这些事件包括但不限于 ：
• 更改密码
• 启用或停用触控 ID 或面容 ID
• 添加或移除触控 ID 指纹或面容 ID 面容
• 重设触控 ID 或面容 ID
• 添加或移除 Apple Pay 卡片
• 抹掉所有内容和设置

在未配备安全储存组件的架构上， EEPROM （电可擦除可编程只读存储器） 被用于为安全隔区提供安全储存服务。 跟安全储存组件类似， EEPROM 连接到安全隔区并仅可从安全隔区访问， 但其不包含专用的硬件安全性功能， 不能确保对熵的独有访问权限 （除了其物理连接特性）， 也不具备计数器加密箱功能。

### Effaceable Storage - 可擦除存储器

NAND 存储器中一个用于储存加密密钥的专用区域，可被直接寻址和安全擦除。尽管当攻击者实际占有设备时，可擦除存储器无法提供保护，但其中存储的密钥可用作密钥层级的一部分，用于实现快速擦除和前向安全性。

### Secure Storage Component - 安全储存组件

芯片设计为使用不可更改的 RO 代码、硬件随机数生成器、加密引擎和物理篡改检测。在支持的设备上，安全隔区与用于储存反重放随机数的安全储存组件配对。为了读取和更新随机数，安全隔区和储存芯片采用安全协议来帮助确保对随机数的排他访问。此技术已更迭多代，提供了不同的安全性保证。

#### User keybag - 用户密钥包

用户密钥包是设备常规操作中使用的封装类密钥的储存位置。例如，输入密码后，`NSFileProtectionComplete`会从用户密钥包中载入并解封。 它是储存在 “无保护” 类中的二进制属性列表 (.plist) 文件。

- 对于搭载 A9 之前的 SoC 的设备，该 .plist **文件的内容通过保存在可擦除存储器中的密钥加密**。为了给密钥包提供前向安全性，用户每次更改密时， 系统都会擦除并重新生成此密钥。
    > `BAG1 Key`负责对密钥包的内容加密
- 对于搭载 A9 或后续型号 SoC 的设备，该 .plist 文件包含一个**密钥**，表示密钥包储存在受**反重放随机数**（由安全隔区控制）保护的有锁储存库中。
    > Besides unlocking the device, a passcode or password provides entropy for certain encryption keys.
    > 表明`the passcode entropy`就是 pasccode，而用户密钥包中存储的密钥就是`the lockbox entropy`

安全隔区管理用户密钥包并且可用于查询设备的锁定状态。 仅当用户密钥包中的所有类密钥均可访问且成功解封时， 它才会报告设备已解锁。

#### Device Keybag - 设备密钥包

设备密钥包用来储存用于涉及设备特定操作数据的封装类密钥。

- 配置为共用的 iPadOS 设备有时需要在用户登录前访问凭证 ；因此，需要一个不受用户密码保护的密钥包。
- iOS 和 iPadOS 不支持对用户独有的文件系统内容进行单独加密，这就意味着系统使用来自设备密钥包的类密钥，对文件独有密钥进行封装。而钥匙串则使用来自用户密钥包中的类密钥来保护用户钥匙串中的项目。
- 在配置为单用户使用 （默认配置） 的 iOS 和 iPadOS 设备中，**设备密钥包和用户密钥包是同一个**，并受用户的密码保护。

When stored, the encrypted file system key is additionally wrapped by an “effaceable key” stored in Effaceable Storage **or** using a media key-wrapping key, protected by Secure Enclave anti-replay mechanism.
> EMF Key 的存储方式发生变化？

NSFileProtectionNone: This class key is protected only with the UID, and is kept in Effaceable Storage.
> 就是 DKey，存储在 Effaceable Storage
---

|SoC|内存保护引擎|安全储存区|AES 引擎| PKA|备注|
|:---:|:---:|:---:|:---:|:---:|:---:|
|A8|加密,认证|EEPROM|是|否||
|A9|加密,认证|EEPROM|DPA保护|是||
|A10|加密,认证|EEPROM|DPA保护、可锁定的种子位|操作系统绑定密钥||
|A11|加密,认证,反重放| EEPROM|DPA保护、可锁定的种子位|操作系统绑定密钥||
|A12|加密,认证,反重放|安全储存组件(第一、二代)| DPA保护、可锁定的种子位| 操作系统绑定密钥|2020年秋季升级|
|A13|加密,认证,反重放|安全储存组件(第一、二代)|DPA 保护、可锁定的种子位|操作系统绑定密钥和启动监视器|2020年秋季升级|

## 参考文献
