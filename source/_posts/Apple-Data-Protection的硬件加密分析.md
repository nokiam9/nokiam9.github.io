---
title: Apple Data Protection的硬件加密分析
date: 2024-03-04 21:23:05
tags:
---

根据 Apple 官网提供的[第三方平台认证信息](https://support.apple.com/zh-cn/guide/certifications/welcome/web)，其硬件、操作系统、应用软件、互联网服务和 Apple Pay 广泛采用 FIPS 140-2 加密标准（FIPS 140-3 认证已提交但尚未通过），当前[硬件加密模块 v10.0](https://csrc.nist.gov/CSRC/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp3811.pdf) 获得 CMVP 颁发的[3811 证书](https://csrc.nist.gov/Projects/cryptographic-module-validation-program/Certificate/3811)。

同时，Apple 也积极进行通用标准 (CC) 评估，当前[符合产品](https://www.niap-ccevs.org/Product/index.cfm)包括：macOS 13 Ventura（含[FileVault](https://www.niap-ccevs.org/MMO/Product/st_vid11348-st.pdf)）、iOS 15 & [iOS 16](https://www.niap-ccevs.org/MMO/Product/st_vid11349-st.pdf)；iPadOS 15 & iPadOS 16，由[ATSEC 公司](https://www.atsec.com)出具 TOE（Target of Evaluation）评估报告。

本文就是依据上述公开信息的研究分析。

## 一、总体架构

根据[iOS 16 安全评估报告](Apple_iOS_16_iPhone_Security_Target_v1.1.pdf)的表述（P16-P20），其提供的加密服务分为三个层级：

- User Space：SL1，软件级。驻留在用户空间内的动态可加载库，为 App 提供加密功能
- Kernel Space：SL1，软件级。操作系统内核加载的扩展模块，仅为内核提供加密功能
- Secure Key Store：SL2，硬件级，也称为 SKS（安全密钥存储）。一个单芯片独立硬件加密模块（即安全隔区），为传输中和静态数据提供保护，包含固件和硬件加密算法实现

![SKS](SKS-arch.png)

Apple 硬件加密服务的系统边界如上图，核心组件包括：

- 区别于 REE 环境的应用处理器 AP，TEE 环境的安全隔区有独立的应用处理器称为 SEP（Secure Encalve Processor），其运行的操作系统 sepOS 是 Apple 基于 L4 微内核的定制版本；TOE OS 和 sepOS 实现物理隔离，仅能通过 Mailbox 完成数据交互
- 安全隔区为加密静态数据所需密钥的安全生成和储存提供基础，必须确保不能透露其用于建立加密密钥关系的 CSP（Critical Security Parameter，关键安全参数），为此内置了 SKS（Secure Key Store，安全密钥存储）模块，并按照 FIPS 140-2 的要求提供 POST（Power-On Self Test，启动自检）功能，以确保模块的完整性和加密功能的正确性
- 在 SoC 生产环节，使用基于 AES 算法的 DRBG 生成 UID，然后蚀刻到硅片上的 OTP-ROM（One Time Programmable Read-Only Memory，一次性编程的只读内存，即熔丝），它只能由 SKS 通过**专用的 AES 硬件引擎**执行加密和解密操作，即使 SEP 的 Apps 也无法读取

对比下图，可以进一步理解逻辑架构和物理架构的关系。
![SoC](soc.png)

## 二、安全存储组件

根据之前的 iOS4 破解技术分析，Apple 的核心密钥有三种存储方式，即：
![arch](arch3.jpg)

- SEP 内部：每个设备唯一且不可修改的UID，基于**固定参数**衍生的`Key 0x89B`和`Key 0x835`
- 可擦除区域（effaceable storage）：`EMF Key`，`Dkey`和`Bag1`。
    以包裹状态的密文存储在 NAND 闪存的 Block 1
- 用户密钥包（User Keybag）：各类文件保护 Class key 和钥匙串 Class key。
    以包裹状态的密文存储在 .plist二进制文件

基于[iOS 16 安全评估报告](Apple_iOS_16_iPhone_Security_Target_v1.1.pdf)，P125 给出了 SKS 密钥层次的官方描述，其密钥层次关系保持不变，但 Class key 不再保存在 User keybag 文件，而是转移到**安全储存组件**!!!

### 1. 核心组件

请参考[Apple平台安全保护-2022](https://help.apple.com/pdf/security/zh_CN/apple-platform-security-guide-cn.pdf)，P61 对于 User Keybag 的描述：

>- 对于搭载 A9 之前的 SoC 的设备，该 .plist 文件的内容通过保存在可擦除存储器中的密钥（**即Bag1**）加密。为了给密钥包提供前向安全性，用户每次更改密码时，系统都会擦除并重新生成此密钥。
>- 对于搭载 A9 或后续型号 SoC 的设备，该 .plist 文件包含一个密钥（**即安全储存组件生成，并返回给安全隔区的加密箱熵值**），表示密钥包储存在受反重放随机数 (由安全隔区控制) 保护的有锁储存库（**即安全存储组件**）中。
>- 安全隔区管理用户密钥包并且可用于查询设备的锁定状态。仅当用户密钥包中的所有类密钥均可访问且成功解封时，它才会报告设备已解锁。

安全隔区技术架构中的 Secure Nonvolatile Storage 翻译为“安全非易失性存储器”，也就是安全存储组件。该报告的 P137 提供了安全存储组件的定义。

- 不可更改的 ROM 代码
- 硬件随机数生成器
- **每个设备唯一的加密密钥**
- 加密引擎
- 物理篡改检测
- 计数器加密箱（第二代增加）：包含 salt、verify、counter、max_retry_limit

### 2. 演进历史

P13 做了进一步的介绍：

>- 安全隔区配备了专用的安全非易失性存储器设备。安全非易失性存储器通过专用的 I2C 总线与安全隔区连接，因此它仅可被安全隔区访问。所有用户数据加密密钥植根于储存在安全隔区非易失性存储器中的熵内。
>- 搭载 A12、 S4 及后续型号 SoC 的设备并用了安全隔区与安全储存组件来储存熵。安全储存组件本身设计为使用不可更改的 ROM 代码、硬件随机数生成器、每个设备唯一的加密密钥、加密引擎和物理篡改检测。安全隔区和安全储存组件使用加密且认证的协议通信以提供对熵的独有访问权限。
>- 2020 年秋季或之后首次发布的设备配备了第二代安全储存组件。第二代安全储存组件增加了计数器加密箱。每个计数器加密箱储存一个 128 位盐、一个 128 位密码验证器、一个 8 位计数器，以及一个 8 位最大尝试值。对计数器加密箱的访问通过加密且认证的协议来实现。

P14 安全隔区的发展历史也包含了安全存储组件的内容。

- A8 ～ A11 的安全存储是基于 EEPROM（电可擦除可编程只读存储器）芯片，只能提供安全存储服务；
- A12 安全存储组件集成了 ROM 代码和若干专用硬件，可以提供完整的硬件安全性功能，并确保对熵的独有访问权限（专用 I2C bus）；
- 2020年为抵御重放攻击紧急开发了第二代产品，关键是增加了计数器加密箱和 xART 机制。

## 三、密钥层级结构

[Apple平台安全保护-2022](https://help.apple.com/pdf/security/zh_CN/apple-platform-security-guide-cn.pdf)指出：
> 文件的内容可能使用文件独有 (或范围独有) 的一个或多个密钥进行加密，密钥使用类密钥封装并储存在文件的元数据中，文件元数据又使用文件系统密钥进行加密。
> 类密钥通过硬件 UID 获得保护，而某些类的类密钥则通过用户密码获得保护。
> 此层次结构既可提供灵活性，又可保证性能。例如，更改文件的类只需重新封装其文件独有密钥，更改密码只需重新封装类密钥。

基于[iOS 16 安全评估报告](Apple_iOS_16_iPhone_Security_Target_v1.1.pdf)，其密钥层级关系如下：

![TOC](<TOC-key.png>)

### 1. 类密钥 - Class key

|Keychain的数据保护类型|File的数据保护类型|适用场景|
|:---:|:---:|:---:|
|kSecAttrAccessibleWhenUnlocked|NSFileProtectionComplete|未锁定状态|
|N/A |NSFileProtectionCompleteUnlessOpen|锁定状态|
|kSecAttrAccessibleAfterFirstUnlock|NSFileProtectionCompleteUntilFirstUserAuthentication |首次解锁后|
|kSecAttrAccessibleAlways|NSFileProtectionNone|始终|

- 三个钥匙串类都有对应的`ThisDeviceOnly`项目，后者在备份期间从设备拷贝时始终通过 UID 加以保护，因此如果恢复至其他设备将无法使用，例如 VPN 证书不适合迁移至另一台设备
- 文件的 Class B 使用了非对称加密算法，钥匙串不提供相应的 Class key。如果 APP 确实存在后台更新的需求，官方建议使用`kSecAttrAccessibleAfterFirstUnlock`，即对应 Class C
- `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly`仅存在于系统密钥包中，仅当设备配置了密码时可用，其行为方式与`kSecAttrAccessibleWhenUnlocked`相同，且它不同步到 iCloud 钥匙串、不会备份、不包括在托管密钥包中。

### 2. 密钥包 - keybag

文件和钥匙串数据保护类的密钥均在密钥包中收集和管理。

User keybag 是设备常规操作中使用的封装类密钥的储存位置（与 passcode 相关），设备密钥包用来储存用于涉及设备特定操作数据的封装类密钥（仅与 UID 相关），两者合称为系统密钥包。
**Secure Key Store** 负责管理系统密钥包，并且可以查询设备的锁定状态，只有当系统密钥包中的所有类密钥都可以访问并且已成功解开包装时，设备才会解锁。
> 配置为单用户使用 (默认配置) 的 iOS 和 iPadOS 设备中，设备密钥包和用户密钥包是同一个

此外，还有用于备份、托管和 iCloud 云备份的几种的密钥包，item 包含了一些其他密钥，但数据结构完全相同。请参考[https://github.com/russtone/systembag.kb](https://github.com/russtone/systembag.kb)。

- Header 包含了 HMCK（校验值）、SALT（PBKDF2的盐）、ITER（迭代次数）等关键参数
- 类密钥 item 的字段`WRAP`定义了该密钥的包裹方式：
  - 1：只有 Dkey 包裹
  - 2：只有 PDK 包裹，未使用！
  - 3：基于`Dkey XOR PDK`包裹。注意，iOS 4 软件加密的实现方式是两次解封，硬件加密的处理逻辑更简单！
- CLAS[1/2/3]分别对应文件保护的 Class A/B/C，CLAS[5]保留未启用！
- CLAS[4]缺失，因为 Class D 的类密钥就是 `Dkey`！
- CLAS[2] 有个额外的字段`PBKY`，保存了 Class B 类密钥的**私钥**
- CLAS[6/7/8/9/10/11]分别对应钥匙串的 3 个级别和相应的 device only

### 3. 关键加密参数 CSP

[Apple SKS 加密模块（非专用设备）安全策略](Apple_Secure_Key_Store_Cryptographic_Module_with_Notes.pdf)的 P20，提供了关键加密因子的列表。
[TOE 评估报告 - iOS 16](Apple_iOS_16_iPhone_Security_Target_v1.1.pdf)的 P119 - P121，介绍了密钥管理的基本概念，也提供了 CSP 的列表信息。

#### 用于快速擦除的三个原生密钥

存储在 Effaceable Stroage 的三个**原生**密钥，可被 sepOS 直接寻址和安全擦除。
尽管当攻击者实际占有设备时，可擦除存储器无法提供保护，但其中存储的密钥可用作密钥层级的一部分，用于实现快速擦除和前向安全性。

- File system object DEK：= `EMF key / Lwvm`，也称 the file system key。
    用于文件系统的元数据加密，基于`Key 0x89B`以 AES-256 KW 封装，密文持久化存储
- Class D Key：= `Dkey`，也称为 the device key。
    用于类密钥`NSFileProtectionNone`的包裹密钥，基于`Key 0x835`以 AES-256 KW 封装，密文持久化存储
- Module-managed keybags key：= `Bag1`
    用于用户密钥包内容的加密/解密，基于`xxx`以 AES-256 KW 封装，密文持久化存储？// TODO:

#### Root Encryption Key (REK)

REK 就是`Passcode-derived key`！
[Apple SKS 加密模块（非专用设备）安全策略](Apple_Secure_Key_Store_Cryptographic_Module_with_Notes.pdf)的 P16 指出，系统服务 Create REK 的关键加密因子包括：

- UID：唯一的 Root ID
- PBKDF Password：：用户输入的登录口令 passcode
- PBKDF Salt for REK：系统密钥包中的盐，随机数
- DRBG internal state：DRBG 的内部状态，包括：V value, key and seed material
- Entropy input string：从 TRNG 获取的真随机数
- REK：？                               // TODO:
- KEK as AES key used to wrap REK：？   // TODO:

#### NVM storage controller shared key

AES 引擎使用的**临时**封装密钥。

每台包含安全隔区的 Apple 设备还具有专用的 AES256 加密引擎 （“AES 引擎”），它内置于 NAND（非易失性）闪存与主系统内存之间的直接内存访问 (DMA) 路径中，可以实现高效的文件加密。在 A9 或后续型号的 A 系列处理器上，闪存子系统位于隔离的总线上，该总线被授权只能通过 DMA 加密引擎访问包含用户数据的内存。

sepOS 启动时使用 TRNG 生成，并通过**专用线路**传输到 AES 引擎，旨在防止它被安全隔区外的任何软件访问。
sepOS 随后可以使用临时封装密钥来封装文件密钥，供应用程序处理器文件系统驱动程序使用。当文件系统驱动程序读取或写入文件时，它会将封装的密钥发送到 AES 引擎以解封密钥。

### 4. 核心处理流程

在[TOE 评估报告 - iOS 16](Apple_iOS_16_iPhone_Security_Target_v1.1.pdf)的 P126，介绍了几个核心流程。

#### TOE 启动

1. 使用安全隔区的 TRNG 在 SEP 中创建一个 256 位的`临时封装密钥`
2. OS 内核从 Effaceable Stroage 读取**包裹状态**的 `Dkey` 和 `EMF key`，并发送至 SEP
3. SEP 持有`Key 0x835`解封`Dkey` ，持有`Key 0x89B`解封 `EMF key`
4. SEP 持有 Step 1 生成的`临时封装密钥`包裹`Dkey`       // TODO:
5. SEP 通过专用线路（I2C协议）将`临时封装密钥`传输到 AES 引擎。OS 内核无法访问此区域。

#### TOE OS 读取文件

1. OS 内核首先提取**加密状态**的文件 meatdata 并将其发送到 SEP
2. SEP 持有`EMF Key`解密文件 meatdata，并将其发送回 OS 内核
3. OS 内核分析文件 meatdata 确定数据保护类别，并将**包裹状态**的`Class Key` 和`per-file key`一起发送到 SEP
    - Class D 的类密钥基于`Dkey`包裹，其他类密钥基于`Dkey XOR Passcode Key`包裹
    - `per-file key`基于指定的`Class Key`包裹
4. SEP 持有`Dkey`和`Passcode Key`解封`per-file key`，使用`临时封装密钥`重新包裹它，并发送回 OS 内核
5. OS 内核将文件访问请求（读取或写入）和**重新包裹状态**的`per-file key`一起发送到存储控制器
6. 存储控制器使用 AES 引擎持有的`临时封装密钥`解封`per-file key`，然后在数据从闪存传输/传入闪存期间解密（读取操作时）或加密（写入操作时）

## 四、MacOS Filevault 的技术分析

基于[TOE 评估报告 - MacOS 13：FileVault](Apple_MacOS_13_Ventura_FileVault_Security_Target.pdf)，P9 指出：

![M1](filevault1.png)
![Intel](filevault2.png)

Apple T2安全芯片运行T2OS 13操作系统。T2包括安全飞地，其中包含运行sepOS操作系统的SEP，以及执行存储加密的DMA存储控制器。加密引擎（EE）在T2上实例化。AA在英特尔芯片（密码获取）和T2上都实例化。安全飞地为所有EE功能（即存储数据的加密/解密除外）和AA的所有加密功能（即PBKDF2）提供安全相关功能。DMA存储控制器提供了一个专用的AES加密引擎，内置在主机平台的存储和主内存之间的直接内存访问（DMA）路径中。密码获取组件（AA）是存储驱动器上的预引导组件。它捕获用户密码并将其传递给安全飞地。

When stored, the encrypted file system key is additionally wrapped by an “effaceable key” stored in Effaceable Storage **or** using a media key-wrapping key, protected by Secure Enclave anti-replay mechanism.

待续！

## 五、总结和分析

### 1. 关于`Key 0x89B`和`Key 0x835`的讨论

在[TOE 评估报告 - iOS 16](Apple_iOS_16_iPhone_Security_Target_v1.1.pdf)的 P117 说明：
> The UID is used to derive two other keys, called "Key 0x89B" and "Key 0x835". These two keys are derived during **the first boot** by encrypting defined constants with the UID.

P121 的密钥列表中，关于这两个 UID 衍生密钥的存储方式描述是：
> SEP. Block 0 of the flash memory. (Effaceable storage.)

笔者认为，这两个 UID 衍生密钥应该是安全隔区**每次**启动时，基于固定值计算得出并保存在内存中，无需持久化存储。

- `Key 0x835 = AES_KW(UID, 0x’01010101010101010101010101010101’)`
- `Key 0x89B = AES_KW(UID, 0x’183e99676bb03c546fa468f51c0cbd49’)`

### 2. TOE 核心流程的几个错误？

关于 TOE 的启动流程，
> The SEP **wraps the Dkey with the newly generated ephemeral key**.

关于文件的存取流程，
> The TOE OS kernel determines which class key to use and **sends the class key** (which is wrapped with the Dkey, or with the XOR of the Dkey and the Passcode Key) and the file key (which is wrapped with the class key) to the SEP.

### 3. 关于 KEK as AES key used to wrap REK 的疑问

这是一个什么东东？

---

## 附录一：名词解释

### 功能组件类

- Effaceable Storage（可擦除存储器）：NAND存储器中一个用于储存加密密钥的专用区域，可被直接寻址和安全擦除。尽管当攻击者实际占有设备时，可擦除存储器无法提供保护，但其中存储的密钥可用作密钥层级的一部分，用于实现快速擦除和前向安全性。
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

## 附录二：Cocoa 框架

Cocoa 是苹果公司为 macOS 所创建的原生面向对象的应用程序接口，是 Mac OS X 上五大 AP 之一（其它四个是Carbon、POSIX、X11和Java）。

Cocoa 应用程序一般在苹果公司的开发工具 Xcode（前身为 Project Builder ）和 Interface Builder 上用Objective-C 写成。

Cocoa包含三个主要的Objective-C对象库，称为“框架”。框架的功能类似于动态库，即可以在运行时动态的加载应用程序的地址空间，但框架作为一个捆绑而非独立文件，其中除了可执行代码外，也包含了资源，头文件和文档。

- Foundation：面向对象的通用函数库。提供了字符串，数值的管理，容器及其枚举，分布式计算，事件循环，以及一些其它的与图形用户界面没有直接关系的功能，其中用于类和常数的函数有`NS`前缀（因为源自于NeXTSTEP）。
    它可以在 MacOS 和 iOS 中使用。
- AppKit（Application Kit，应用程序工具包）：包含了程序与图形用户界面交互所需的代码。基于 Foundation 创建的，也使用`NS`前缀（因为其直接派生自 NeXTSTEP）。
    它只能在 MacOS 中使用。
- UIKit（User Interface Kit，用户界面工具包）：用于 iOS 的图形用户界面工具包。与AppKit不同，它使用`UI`的前缀。

---

## 参考文献

- [iOS 取证技术 - elcomsoft blog](https://blog.elcomsoft.com/2023/03/perfect-acquisition-part-1-introduction/)
  
### 官方文档下载

- [Apple 平台安全保护 - 2022年中文版](apple-platform-security-guide-cn-2022.pdf)
- [Apple SKS 加密模块（非专用设备）安全策略 - v10注释版](Apple_Secure_Key_Store_Cryptographic_Module_with_Notes.pdf)
- [TOE 评估报告 - MacOS 13：FileVault](Apple_MacOS_13_Ventura_FileVault_Security_Target.pdf)
- [TOE 评估报告 - iOS 16](Apple_iOS_16_iPhone_Security_Target_v1.1.pdf)
- [Apple T2 安全芯片概览](Apple_T2_Security_Chip_Overview_2018.pdf)

### 研究报告下载

- [iOS Platform Security](Platform_Security.pdf)
- [Behind the Scenes with iOS Security](Behind_the_Scenes_with_iOS_Security.pdf)
- [Data Security on Mobile Devices:Current State of the Art, Open Problems, and Proposed Solutions](Data_Security_on_Mobile_Devices.pdf)
