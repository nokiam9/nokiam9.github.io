---
title: Apple Secure Key Store 硬件加密技术分析
date: 2024-04-07 21:08:41
tags:
---

根据[iOS 16 安全评估报告](Apple_iOS_16_iPhone_Security_Target_v1.1.pdf)的表述（P16-P20），Apple 设备提供的加密服务分为三个层级：

- User Space：SL1，软件级。驻留在用户空间内的动态可加载库，为 App 提供加密功能
- Kernel Space：SL1，软件级。操作系统内核加载的扩展模块，仅为内核提供加密功能
- Secure Key Store：SL2，硬件级。一个单芯片独立硬件加密模块，为静态数据和数据传输提供保护，包含固件和硬件加密算法实现

Secure Key Store（安全密钥存储，SKS，也称 Secure Key Service），是 Apple 公司基于 Secure Enclave 的加密模块，也是其硬件加密技术的核心组件。

- 根据 Apple 官网提供的[第三方平台认证信息](https://support.apple.com/zh-cn/guide/certifications/welcome/web)，其硬件、操作系统、应用软件、互联网服务和 Apple Pay 广泛采用 FIPS 140-2 加密标准（FIPS 140-3 认证已提交但尚未通过），当前[硬件加密模块 v10.0](https://csrc.nist.gov/CSRC/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp3811.pdf) 获得 CMVP 颁发的[3811 证书](https://csrc.nist.gov/Projects/cryptographic-module-validation-program/Certificate/3811)。
- Apple 积极参与通用标准 (CC) 评估，当前[认证产品清单](https://www.niap-ccevs.org/Product/index.cfm)包括：macOS 13 Ventura（含[FileVault](https://www.niap-ccevs.org/MMO/Product/st_vid11348-st.pdf)）、iOS 15 & [iOS 16](https://www.niap-ccevs.org/MMO/Product/st_vid11349-st.pdf)；iPadOS 15 & iPadOS 16，由 [ATSEC 公司](https://www.atsec.com)出具 TOE（Target of Evaluation）评估报告。
- 2016年美国举办的 Black hat 会议上，Apple 公司的 Ivan Krstic 介绍了数据保护技术，也是难得的[官方资料]((Behind_the_Scenes_with_iOS_Security.pdf))。

本文就是依据上述公开信息的研究分析。

## 一、Secure Key Store 硬件环境

根据[Apple SKS 加密模块](Apple_Secure_Key_Store_Cryptographic_Module_with_Notes.pdf)的表述，其加密模块的逻辑边界包含组件：

- SEP（Secure Encalve Processor）：区别于 REE 环境的应用处理器 AP，位于 TEE 环境的安全隔区拥有独立的应用处理器
    SEP 运行的操作系统称为 sepOS（基于 L4 微内核的 Apple 定制版本）
    TOE OS 和 sepOS 之间采用物理隔离方式，仅能通过 Mailbox 完成数据交互
- SKS（Secure Key Store，安全密钥存储）：负责生成并管理 CSP（Critical Security Parameter，关键安全参数），为静态数据加密所需密钥提供关键的加密材料
    按照 FIPS 140-2 的要求提供 POST（Power-On Self Test，启动自检）功能，以确保模块的完整性和加密功能的正确性
- HW AES：基于硬件实现的 AES 加密算法，Apple 安全隔区支持对抗时序攻击、静态功耗分析（SPA）和动态功耗分析 (DPA)。
- UID：在 SoC 生产环节，使用基于 AES 算法的 DRBG 生成 UID，然后蚀刻到硅片上的 OTP-ROM（One Time Programmable Read-Only Memory，一次性编程的只读内存，即熔丝），它只能通过 AES 引擎处理加密和解密操作，即使 SEP 的 Apps 也无法读取
- HW DRBG：基于硬件实现的伪随机数发生器（确定性随机数发生器），内置专用的 AES 引擎实现 CTR_DRBG 算法

![SKS](SKS-arch.png)

> 早期的 A8 处理器的硬件版本是 1.0，A9 及后续处理器升级为 2.0，差异在于是否提供 PAA (Processor Algorithm Accelerator，处理器算法加速，基于 ARM NEON 指令集实现)。

对比 Apple 平台安全技术白皮书提供的信息，HW AES = Secure Enclave AES Engine，此外还有 PKA（public key acceleration，公钥加速器）。

![SoC](soc.png)

## 二、 Secure Key Store 密码算法

[Apple SKS 加密模块](Apple_Secure_Key_Store_Cryptographic_Module_with_Notes.pdf)有完整的的表述。

### 1. 完全符合 FIPS 140-2 标准的算法

- 对称加密/解密算法：基于[FIPS 197](https://csrc.nist.gov/pubs/fips/197/final)的 AES 算法，工作模式包括：
  - [SP 800-38 A](https://csrc.nist.gov/pubs/sp/800/38/a/final) ：工作模式 ECB CBC CFB OFB CTR
  - [SP 800-38 C](https://csrc.nist.gov/pubs/sp/800/38/c/upd1/final) ：加密认证模式 CCM
  - [SP 800-38 D](https://csrc.nist.gov/pubs/sp/800/38/d/final) ：工作模式 GMAC
  - [SP 800-38 E](https://csrc.nist.gov/pubs/sp/800/38/e/final) ：磁盘加密模式 AES-XTS
  - [SP 800-38 F](https://csrc.nist.gov/pubs/sp/800/38/f/final) ：密钥包裹传输 AES-KW
- 数字签名（非对称加密/解密）算法：基于[FIPS 186-4](https://csrc.nist.gov/pubs/fips/186-4/final)，
  - ECDSA 椭圆曲线采用 P-2224, P-256, P-384, P-521，加密算法符合 ANSI X9.62 标准
  - 签名生成和校验算法采用 SHA
- 消息摘要算法：基于[FIPS 180-4](https://csrc.nist.gov/pubs/fips/180-4/upd1/final) 的 SHA 算法
- 消息验证算法：基于[FIPS 198](https://csrc.nist.gov/pubs/fips/198-1/final)的 HMAC 算法
- DRBG：基于[SP 800-90A](https://csrc.nist.gov/pubs/sp/800/90/a/r1/final) 的确定性随机数生成算法
- 密钥交换算法：基于[SP 800-56A](https://csrc.nist.gov/pubs/sp/800/56/a/r3/final) 的一次性 DH 算法

> ANSI（American National Standards Institute，美国国家标准协会）是一个非盈利组织，也是 IEEE 和 ISO 标准化组织的成员，还记得 ANSI C 吗？
> [ANSI X9](https://x9.org/) 系列标准主要用于金融服务业，是信息和数据安全的重要参考。

### 2. 不完全符合 FIPS 140-2 标准，但被允许使用的算法

- 密钥扩展算法：基于[SP 800-132](https://csrc.nist.gov/pubs/sp/800/132/final)的 PBKDF 算法，Apple 基于 HMAC 和 SHA-256（早期为 SHA-1）的定制化 PBKDF2 算法，注意不支持自检功能！
- TRNG：Apple 定制的，基于物理过程的真随机数生成器

### 3. 未获得 FIPS 140-2 批准的算法

对于此类算法，Apple 规定**不得与上述算法共享加密因子**。

- 基于 RSA 的密钥包裹协议：验证配件、VPN 连接和 iMessage 等场景中使用。填充方案采用 OAEP（[Optimal Asymmetric Encryption Padding](https://en.wikipedia.org/wiki/Optimal_asymmetric_encryption_padding)，最佳非对称加密填充）
- ECDH 密钥交换协议：使用了未被批准的 Curve25519 曲线，而且使用了密钥模块外部的加密因子。用于文件保护 B 类密钥管理、iCloud 云备份密钥管理、HomeKit 配件通信等场景。
- Ed25519 签名方案：使用了未被批准的 Curve25519 曲线。
- [RFC 5869 HKDF](https://datatracker.ietf.org/doc/html/rfc5869)：使用了未被批准的、基于 HMAC 的密钥扩展算法。HomeKit 配件连接和 iMessage 等场景中使用。
- [ANSI X9.63]((https://csrc.nist.gov/CSRC/media/Events/Key-Management-Workshop-2000/documents/x963_overview.pdf)) KDF：用于金融服务行业密钥协议和密钥传输，同样采用 Curve25519
- AES-GCM：IV 的生成方式不符合 FIPS 140-2

> 事实上，Curve25519 已成为 P-256 的替代品。2024 年，NIST 宣布弃用 FIPS 186-4 并发布[FIPS 186-5](https://csrc.nist.gov/pubs/fips/186-5/final)，Curve25519 和 Curve448 被正式纳入规范，ANSI X9.62 也被 ANSI X9.142 替代

### 4. PBKDF2 算法

用户的锁屏密码 Passcode 的长度有限，不能直接作为密钥使用，必须经过密钥扩展（Password-Based Key Derivation）生成标准的 Passcode Key，参考[SP 800-132](https://csrc.nist.gov/pubs/sp/800/132/final)第 5.4 节的 2b 模式。

基于[Behind the Scenes with iOS Security - Ivan Krstić](Behind_the_Scenes_with_iOS_Security.pdf)的描述，其处理步骤包括：

- 初始阶段：Userspace 侧完成。使用 HMAC-SHA-256 作为伪随机函数
    输入：基于TRNG 生成的 128 位随机 salt、未经任何预处理的 passcode、**迭代计数 1**
    输出：256 位的中间密钥 MK
- 迭代阶段：SEP 侧完成。以 UID 为加密密钥，使用 AES-CBC-256 硬件重复迭代加密 MK，最终形成 Master Key
    迭代次数需要根据硬件环境校准，确保计算时间为 100-150 毫秒，迭代次数至少为 50000 次

> 大量重复运算使得暴力破解的成本很高，而添加盐值可以有效防御彩虹表攻击。

![KDF](KDF.png)

需要注意的是，不同版本硬件的算法和配置都可能发生变化：

- iOS 4 迭代算法与标准规范有差异，增加了将 256位 MK 扩展为 4096 位的环节
- iOS 4.3 升级了 Keybag 版本，v2 算法的 counter 更复杂
- Filevault 软件加密方式的迭代次数为 41000，但加密因子不包含 UID！

![PBKDF2](PBKDF2.png)

## 三、Secure Key Store 密钥层级

根据之前的 iOS4 破解技术分析，Apple 数据保护密钥的持久化存储有 4 种方式：

- SEP 内部：每个设备唯一且不可修改的UID，基于固定参数衍生的Key 0x89B 和 Key 0x835
- 可擦除区域（effaceable storage）：EMF Key，Dkey 和 Bag1
    包裹状态密文保存在 NAND 的 Block 1，不提供额外的机密性保护，而是便于快速/远程清除
- 用户密钥包（User Keybag）：各类文件保护 Class key 和钥匙串 Class key
    包裹状态密文保存在一个普通的数据文件（plist格式），存储于`/var/keybags/systembag.kb`
- per-file key 以包裹状态的密文存储在文件系统元数据中，keychain 的 item 以包裹状态保存在一个 sqlite 数据库，存储目录位于`/Users/<UserName>/Library/Keychains/`

![arch](arch3.jpg)

> 注意，系统密钥包**从不存储 Class D key**！因为 NSFileProtectionNone 实际是一个单密钥系统，该类所有文件的密钥完全相同，即：`Class D key = AES_Unwrap(key 0x835，Dkey)`

### 1. 关键加密因子

[iOS 16 安全评估报告](Apple_iOS_16_iPhone_Security_Target_v1.1.pdf)的 P119 - P121，介绍了密钥管理的基本概念并提供了 CSP（Critical Security Parameter，关键安全参数）的列表信息，P125 提供了密钥层次结构图，与 iOS4 的描述基本一致，

![TOC](<TOC-key.png>)

[Apple SKS 加密模块](Apple_Secure_Key_Store_Cryptographic_Module_with_Notes.pdf)的 P20，也提供了关键安全参数的生命周期信息：

1. Root Encryption Key (REK) = Passcode Key，其描述为
    > Generation：Derived from passcode using PBKDF2 and entanglement with UID
    > Entry and Output：Generated inside the module and is never output。
2. CSP 列表的 Moudle-managed keybags key 所指**并非 Bag1**(因为其是明文存储方式)，而是密钥包中存储的各种 Class key
    > Generation: Symmetric key generation services of the module
    > Entered encrypted using AES-256 KW, Output encrypted using AES-256 KW
    > Non-volatile store: cryptographically zeroized when overwriting the KEK
    > Volatile store: zeroized when freeing the secure memory
3. CSP 列表的 File system object DEK = EMF
4. CSP 列表的 Escrow Keybag and the class D key encryption key = Key 0x835，
   但没有提供 Key 0x89B 的信息

> CSP 列表的 REK wrapping key = SKS 和 SBIO 之间的 random secret，分析见后文！！！

### 2. 密钥包

文件和钥匙串数据保护类的密钥均在密钥包中收集和管理。**Secure Key Store**负责管理系统密钥包，并且可以查询设备的锁定状态，只有当系统密钥包中的所有类密钥都可以访问并且已成功解开包装时，设备才会解锁。

User keybag 是设备常规操作中使用的封装类密钥的储存位置（与 passcode 相关），设备密钥包用来储存用于涉及设备特定操作数据的封装类密钥（仅与 UID 相关），两者合称为系统密钥包。
配置为单用户使用 (默认配置) 的 iOS 和 iPadOS 设备中，设备密钥包和用户密钥包是同一个。

此外，还有用于备份、托管和 iCloud 云备份的几种的密钥包，item 包含了一些其他密钥，但数据结构完全相同。请参考[https://github.com/russtone/systembag.kb](https://github.com/russtone/systembag.kb)。
[Behind the Scenes with iOS Security - Ivan Krstić](Behind_the_Scenes_with_iOS_Security.pdf)的 P27，也给出了一个示例。

![keybag](keybag2.png)

密钥包的头部字段包括：

- VERS：版本号，iOS 12 之后都置为 4
- TYPE：该密钥包的类型，[0-system，1-backup，2-escrow，3-iCloud Backup]
- UUID：该密钥包的 UUID
- HMCK：如果密钥包已签名，保存 HMAC 校验值，即**安全存储组件计算、并返回的加密箱熵值**
- WRAP：包裹方式，[1:仅 UID 派生, 2:仅 passcode 派生, 3:两者都有]
- SALT：PBKDF2 算法使用的盐
- ITER：PBKDF2 算法的迭代次数

> 不同版本可能出现 GRCE、GRNT、TKMT、SART 等字段，信息含义未知

对于每个密钥包的条目（文件保护类密钥 & 钥匙串类密钥），其字段包括：

- CLAS：密钥的级别
    文件保护类型为[1,2,3]：4-空缺，就是Dkey；5-保留未启用；
    钥匙串保护类型为[6,7,8,9,10,11]：对应钥匙串的 3 个级别和相应的 device only
- WRAP：包裹方式，与 Header 定义一致
- KTPY：密钥类型，例如 Curve25519
- WPKY：处于包裹状态的类密钥数据。如果是非对称密钥，此处存储的是 Private key
- PBKY：可选（例如 CLASS B）。非对称密钥对的 Pulic key，包裹状态
- UUID：该密钥的 UUID

关于密钥包的解封流程，[Apple SKS 加密模块](Apple_Secure_Key_Store_Cryptographic_Module_with_Notes.pdf)的 P23 指出：

- 所有在钥匙包中、由加密模块管理的密钥，由 Device OS 负责存储在 non-volatile memory 中。
- 密钥包基于 256 位的 AES-KW 包裹算法，然后输出到 Device OS 完成持久化存储（permanent storage）。
- 设备加电启动后，加密模块从 Device OS 导入包裹状态的密钥包并完成解封。

该模块通过以下方式实现基于密码的身份验证：

- 当用户从模块请求加密服务时，它必须提供密码和对用户密钥包的引用，该密钥包在 SKS 内的 SP800-38F AES 密钥包装 (AES-KW) 下加密存储。
- 该模块使用 PBKDF 从操作者提供的密码中派生 AES 密钥。
- 然后，模块的 SP800-38F AES 密钥解包功能（即 AES-KW-AD）使用派生的 AES 密钥来解密参考用户密钥包并验证解密密钥的真实性。
- 由于 AES-KW 是一种身份验证密码，因此解密操作只有在没有身份验证错误的情况下才会成功。这意味着用户提供了正确的密码来导出用于 AES 密钥解包的正确 AES 密钥。
- 任何其他密码都将派生出不同的 AES 密钥，这将导致解密的用户密钥错误，导致身份验证检查失败。
- 如果可以成功解开用户密钥包，则用户将通过模块的身份验证，然后将使用解开的用户密钥继续执行所请求的加密服务。
- 解包用户密钥包失败也是用户身份验证失败，操作员将被拒绝访问模块。

## 四、Secure Key Store 业务逻辑

根据[Demystifying the Secure Enclave Processor](us-16-Mandt-Demystifying-The-Secure-Enclave-Processor.pdf)的 P61 的描述，Secure Key Store 作为 SEPOS 的一个功能模块，其暴露在 SL1 级（内核 XNU）的 Endpoint 就是一个内核扩展`AppleKeyStore.kext`。此外，SEPOS 还包含了其他类型的 APP，包括：

- ART Manager：反重放攻击服务
- Secure Biometric Engine：生物识别引擎，简称 SBIO，用于 Touce ID 和 Face ID 等
- Secure Credential Manager：凭据管理，用于网络连接和配件的安全管理
- SSE：未知。似乎用于硬件识别？

![SEPOS](sepos.png)

[iOS 16 安全评估报告](Apple_iOS_16_iPhone_Security_Target_v1.1.pdf)的 P125-P127，介绍了SEP启动、第一次解锁、文件处理等核心业务场景的处理逻辑，结合[Behind the Scenes with iOS Security - Ivan Krstić](Behind_the_Scenes_with_iOS_Security.pdf)的 P31-P34 的流程图，做个简要分析。

### 1. 系统启动

![File](boot.jpg)

- SEP 使用 NDRBG 生成一个`临时封装密钥`，并通过专用线路传送给 NVM 控制器的 AES 引擎
- XNU（操作系统内核）的 AppleKeyStore 从 Effaceable storage 加载包裹状态的 EMF key 和 Dkey
- SEP 持有`Key 0x835`解封`Dkey`，将 Class D key 加入 Keyring
- SEP 使用`临时封装密钥`重新包裹 Class D key，**用于后续该类型文件的统一解密密钥**
- XNU 加载系统分区表和文件系统元数据，发送给 SEP
- SEP 持有`Key 0x89B`解封`EMF key`，持有`media key`解密文件系统元数据，并返回给 XNU

现在 Userspace 就可以正确处理 Class D 类型的文件了，然后是处理密钥包：

- 用户空间启动`keybagd`系统进程，读取无保护的数据文件`/var/keybags/systembag.kb`
- `keybagd`从 Effaceable storage 加载 keybag prot key（即：Bag1，明文），并完成文件级解封
- `keybagd`将解封后的密钥包发送给 SEP，SKS 将 Class B 的 Public key 加入 Keyring

此时，SKS 就可以**新建 Class B 类型文件**（但不能读取已有文件，因为没有 Private key）；此外，SKS 安全内存中还有仍处于包裹状态的 Class A key、Class B 的 Private key 和 Class C key，需要等待第一次解锁！

### 2. 首次解锁

![File](first-unlock.jpg)

- 用户空间的 SpringBoard（用户解锁界面）获得 passcode，通过 XNU 的 AppleKeyStore 发送给 SEP
- SEP 的 Secure Key Store 基于 PBKDF2 算法生成 master key（即 Passcode key 或 REK）
- 基于 master key XOR Dkey 解封密钥包，如果校验成功则将 Boot 阶段未完成的 3 个类密钥加入 Keyring

此时 SKS 就可以正确处理所有类型的文件了，然后是处理 Touch ID：

- SEP 的 SKS 随机生成一个 random secret
- SKS 使用 random secret 包裹 master key，并保存在 SEP 内部
- SKS 将 random secret 发送 SBIO
- SKS 从内存中销毁 master key
  
这样以来，后续在没有输入 passcode 的情况下，可以使用 Touch ID 来恢复类密钥。

### 3. 系统锁定

![File](lock.jpg)

如果用户界面触发了系统锁定事件，SKS 将从 Keyring 中删除 Class A key 和 Class B 的 private key。
注意！此时 Class C key 仍然可用，因为已经成功完成一次解锁。

### 4. 指纹解锁

![File](touch-id-unlock.jpg)

- 用户按压 Home 键时，启动指纹传感器
- 指纹传感器匹配已存储模版，如果检测成功，SEP 的 SBIO 将 random secret 发送给 SKS
- SKS 根据 random secret 解封获得 master key，并将 Class A key 和 Class B 的 private key 添加到 Keyring

现在，SKS 又可以准确处理所有文件了。

### 5. 读取文件

![File](file.jpg)

## 五、 Secure Key Store 服务接口

[Apple SKS 加密模块](Apple_Secure_Key_Store_Cryptographic_Module_with_Notes.pdf)的 P15-P17，提供了 SKS 系统服务清单。

### 1. 完全符合 FIPS 140-2 的系统服务

1. 文件系统服务（Class D File System Services）：无保护类型的文件（读/写）服务
    Key wrapping: UID，AES Key used to wrap Class D Key（key 0x835），Class D Key
    File system keys：DEK（EMF 或 VEK）
    Storage controller key：KEK（ DMA临时密钥 ）
2. 用户密钥包服务（User Keybag Services）：从 Device OS 导入包裹状态的用户密钥包
    Keybag wrapping：AES Key shared with NVM Storage Controller
    Keybag content：KEK
3. 设备密钥包服务（Device Keybag Services）：
    Keybag wrapping: UID,AES Keys as part of module-managed keybags
    Keybag content: KEK
    Keychain keys: DEK
    Sign/verify key: ECDSA Private Key
4. 备份密钥包服务（Backup Keybag Services）：
    Keybag wrapping：AES Key used to wrap Backup Keybag，PBKDF password for Backup Keybag，PBKDF Salt for Backup Keybag
    Keybag content: KEK
    Backup keys: DEK
5. 托管密钥包服务（Escrow Keybag Services）：
    Keybag wrapping：AES Key used to wrap Escrow Keybag
    Keybag content：KEK from User Keybag，DEK
6. iCloud密钥包服务（iCloud Keybag Services）：
    Keybag wrapping：REK derived from UID
    Keybag content：DEK
7. 创建REK（Create REK）：
    加密因子：UID，PBKDF Password，PBKDF Salt for REK，DRBG internal state，Entropy input string，REK，KEK as AES key used to wrap REK
8. 更新REK（Update REK）：
    加密因子：UID，Old/new PBKDF Password，PBKDF Salt for REK，DRBG internal state，Entropy input string，Old/new REK derived from PBKDF Password
9. 生成密钥对（Generate Ref-Keys）：
    加密因子：EC Key Pair，Password，PBKDF Salt for EC Private Key Encryption Key
10. 生成共享密钥（Generate Shared Secret using EC keys generated by the module）：
    加密因子：EC Key Pair，Password，PBKDF Salt for EC Private Key Encryption Key
11. 出厂重置（Erase all content）：清除 UID 之外的所有密钥和关键因子
12. 重启自检（Reboot that implies Self-test）：
13. 显示状态（Show Status）：

### 2. 不符合 FIPS 140-2 的系统服务

1. Generate Shared Secret using EC keys generated outside of the module
2. Ed 25519 Digital signature generation
3. Hash based KDF based on ANSI X9.63
4. EC Diffie-Hellman Key Agreement using curve 25519
5. RFC 5869 based HKDF
6. AES-GCM Encryption and Decryption X

## 六、MacOS Filevault 的技术分析

基于[TOE 评估报告 - MacOS 13：FileVault](Apple_MacOS_13_Ventura_FileVault_Security_Target.pdf)，P9 指出：

![M1](filevault1.png)
![Intel](filevault2.png)

Apple T2安全芯片运行T2OS 13操作系统。

T2包括安全飞地，其中包含运行sepOS操作系统的SEP，以及执行存储加密的DMA存储控制器。
加密引擎（EE）在T2上实例化。AA在英特尔芯片（密码获取）和T2上都实例化。
安全飞地为所有EE功能（即存储数据的加密/解密除外）和AA的所有加密功能（即PBKDF2）提供安全相关功能。

DMA存储控制器提供了一个专用的AES加密引擎，内置在主机平台的存储和主内存之间的直接内存访问（DMA）路径中。密码获取组件（AA）是存储驱动器上的预引导组件。它捕获用户密码并将其传递给安全飞地。

When stored, the encrypted file system key is additionally wrapped by an “effaceable key” stored in Effaceable Storage **or** using a media key-wrapping key, protected by Secure Enclave anti-replay mechanism.

AA - Authorization Acquisition
EE - Encryption Engine

待续！

## 七、总结分析

### 2. 关于 `Key 0x89B` 和 `Key 0x835` 的存储方式

在[TOE 评估报告 - iOS 16](Apple_iOS_16_iPhone_Security_Target_v1.1.pdf)的 P117 说明：
> The UID is used to derive two other keys, called "Key 0x89B" and "Key 0x835". These two keys are derived during **the first boot** by encrypting defined constants with the UID.

P121 的密钥列表中，关于这两个 UID 衍生密钥的存储方式描述是：
> SEP. Block 0 of the flash memory. (Effaceable storage.)

P125
> The UID (a.k.a. UID key) is not accessible by any software. The "Key 0x89B" and "Key 0x835" keys are both derived by encrypting **defined values (identical for all devices)** with the UID key. All three keys are stored in the SEP. All other keys shown in the figure are stored in wrapped form in persistent storage and unwrapped when needed.

笔者认为，这两个 UID 衍生密钥应该是安全隔区**每次**启动时，基于固定值计算得出并保存在内存中，无需持久化存储。

- `Key 0x835 = AES_KW(UID, 0x’01010101010101010101010101010101’)`
- `Key 0x89B = AES_KW(UID, 0x’183e99676bb03c546fa468f51c0cbd49’)`

### 3. 关于 KEK as AES key used to wrap REK

CSP 列表 16 行的`REK wrapping key`，具体描述为：内部原生的对称密钥，Entered/Output in plaintext by calling application within physical the boundary。

笔者判断，这个 KEK 指的是 SKS 和 SBIO 的共享随机密钥，其处理逻辑为：

- 首次输入 passcode 成功解封 keybag 后，SKS 生成该共享密钥作为 KEK
- 基于该密钥包裹 REK（该文称 Master Key）并保存在 SKS 内部
- SKS 将共享密钥发送给 SBIO，并销毁该密钥和 REK
- 如果 touch ID 匹配成功，向 SKS 发送共享密钥；
- SKS 据此解封 REK，进而解封相应的类密钥并添加到 Keyring

### 4. TOE 核心流程的几个错误？

[iOS 16 安全评估报告](Apple_iOS_16_iPhone_Security_Target_v1.1.pdf)的 P126，描述了文件的存取流程。
> The TOE OS kernel determines which class key to use and **sends the class key** (which is wrapped with the Dkey, or with the XOR of the Dkey and the Passcode Key) and the file key (which is wrapped with the class key) to the SEP.

笔者认为，发送给 SEP 的应该是 Class key type，而非包裹状态的 Class key。
iOS 4 破解技术中介绍，WARP 类型‘3’ 是两轮解封，似乎硬件加密将其简化为：`Dkey XOR PDK`

此外，关于 TOE 的启动流程，其描述为：
> The SEP **wraps the Dkey with the newly generated ephemeral key**.

通过临时密钥包裹 Dkey，有何实际意义，目的又是发送给哪个组件呢？

### 5. 关于安全存储组件的讨论？

核心问题是：

1. 安全存储组件是一个独立的、具备存储功能的硬件芯片，还是 NAND 闪存的某个特殊区域？
2. 系统密钥包究竟是存储在文件系统的普通文件，还是 Effaceable Storage 呢？
3. 系统密钥包和 AFPS 的容器密钥包、卷宗密钥包的关系是什么？
4. 安全存储组件的内部到底存储了哪些关键数据？

请参考[Apple平台安全保护-2022](https://help.apple.com/pdf/security/zh_CN/apple-platform-security-guide-cn.pdf)，P9 和 P61 对于 User Keybag 的描述：

>- 虽然安全隔区不含储存设备，但它拥有一套将信息安全储存在所连接储存设备上的机制，该储存设备与应用程序处理器和操作系统使用的 NAND 闪存互相独立。
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

P13 做了进一步的介绍：

>- 安全隔区配备了专用的安全非易失性存储器设备。安全非易失性存储器通过专用的 I2C 总线与安全隔区连接，因此它仅可被安全隔区访问。所有用户数据加密密钥植根于储存在安全隔区非易失性存储器中的熵内。
>- 搭载 A12、 S4 及后续型号 SoC 的设备并用了安全隔区与安全储存组件来储存熵。安全储存组件本身设计为使用不可更改的 ROM 代码、硬件随机数生成器、每个设备唯一的加密密钥、加密引擎和物理篡改检测。安全隔区和安全储存组件使用加密且认证的协议通信以提供对熵的独有访问权限。
>- 2020 年秋季或之后首次发布的设备配备了第二代安全储存组件。第二代安全储存组件增加了计数器加密箱。每个计数器加密箱储存一个 128 位盐、一个 128 位密码验证器、一个 8 位计数器，以及一个 8 位最大尝试值。对计数器加密箱的访问通过加密且认证的协议来实现。

P14 安全隔区的发展历史也包含了安全存储组件的内容。

- A8 ～ A11 的安全存储是基于 EEPROM（电可擦除可编程只读存储器）芯片，只能提供安全存储服务；
- A12 安全存储组件集成了 ROM 代码和若干专用硬件，可以提供完整的硬件安全性功能，并确保对熵的独有访问权限（专用 I2C bus）；
- 2020年为抵御重放攻击紧急开发了第二代产品，关键是增加了计数器加密箱和 xART 机制。

![his](his.png)

[iOS 16 TOE] P126
> The TOE OS kernel determines which class key to use and sends the class key (which is **wrapped with the Dkey**, or **with the XOR of the Dkey and the Passcode Key**) and the file key (which is wrapped with the class key) to the SEP.
> The (wrapped) Dkey and (wrapped) EMF key (both 256-bit keys) are loaded by the TOE OS kernel from the **effaceable storage** and sent to the SEP.

P127
> The EMF key, Dkey, and the class keys are stored in the **effaceable area**, all in wrapped form only.

---

## 附录一：名词解释

### 功能组件类

- Effaceable Storage（可擦除存储器）：NAND存储器中一个用于储存加密密钥的专用区域，可被直接寻址和安全擦除。尽管当攻击者实际占有设备时，可擦除存储器无法提供保护，但其中存储的密钥可用作密钥层级的一部分，用于实现快速擦除和前向安全性。
    A dedicated area of NAND storage, used to store cryptographic keys, that can be addressed directly and wiped securely. While it doesn’t provide protection if an attacker has physical possession of a device, keys held in Effaceable Storage can be used as part of a key hierarchy to facilitate fast wipe and forward security.
- Secure Storage Component（安全储存组件）：一个芯片，设计为使用不可更改的 RO 代码、硬件随机数生成器、加密引擎和物理篡改检测。在支持的设备上，安全隔区与用于储存反重放随机数的安全储存组件配对。为了读取和更新随机数，安全隔区和储存芯片采用安全协议来帮助确保对随机数的排他访问。此技术已更迭多代，提供了不同的安全性保证。
    A chip designed with immutable RO code, a hardware random number generator, cryptography engines, and physical tamper detection. On supported devices, the Secure Enclave is paired with a Secure Storage Component for anti-replay nonce storage. To read and update nonces, the Secure Enclave and storage chip employ a secure protocol that helps ensure exclusive access to the nonces. There are multiple generations of this technology with differing security guarantees.
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

Cocoa 是苹果公司为 macOS 所创建的原生面向对象的应用程序接口，是 Mac OS X 上五大 AP 之一（其它四个是Carbon、POSIX、X11 和 Java）。

Cocoa 应用程序一般在苹果公司的开发工具 Xcode（前身为 Project Builder ）和 Interface Builder 上用Objective-C 写成。

Cocoa包含三个主要的 Objective-C 对象库，称为“框架”。框架的功能类似于动态库，即可以在运行时动态的加载应用程序的地址空间，但框架作为一个捆绑而非独立文件，其中除了可执行代码外，也包含了资源，头文件和文档。

- Foundation：面向对象的通用函数库。提供了字符串，数值的管理，容器及其枚举，分布式计算，事件循环，以及一些其它的与图形用户界面没有直接关系的功能，其中用于类和常数的函数有`NS`前缀（因为源自于NeXTSTEP）。
    它可以在 MacOS 和 iOS 中使用。
- AppKit（Application Kit，应用程序工具包）：包含了程序与图形用户界面交互所需的代码。基于 Foundation 创建的，也使用`NS`前缀（因为其直接派生自 NeXTSTEP）。
    它只能在 MacOS 中使用。
- UIKit（User Interface Kit，用户界面工具包）：用于 iOS 的图形用户界面工具包。与AppKit不同，它使用`UI`的前缀。

## 附录三：侧信道攻击

在密码学中，侧信道攻击（side-channel attack，也称旁路攻击）是一种基于密码系统的**物理实现**中获取信息的攻击方式，区别于暴力破解法，或者基于算法理论性弱点的密码分析。
值得注意的是，如果破解密码学系统使用的信息是通过与其使用人的合法交流获取的，这通常归类于社会工程学攻击。

根据借助的介质，旁路攻击分为多个大类，包括：

- 缓存攻击（Cache attack）：通过获取对缓存的访问权而获取缓存内的一些敏感信息，例如攻击者获取云端主机物理主机的访问权而获取存储器的访问权；
- 计时攻击（Timing attack）：基于测量各种计算（例如，将攻击者的给定密码与受害者的未知密码进行比较）所需的时间的攻击。
- 功耗监控攻击（Power-monitoring attack）：同一设备不同的硬件电路单元的运作功耗也是不一样的，因此一个程序运行时的功耗会随着程序使用哪一种硬件电路单元而变动，据此推断出资料输出位于哪一个硬件单元，进而窃取资料；
    进一步细分为 SPA（simple power analysis，静态功耗分析）和 DPA（differential power analysis，动态功耗分析）
- 电磁攻击（Electromagnetic attack ）：设备运算时会泄漏电磁辐射，经过得当分析的话可解析出这些泄漏的电磁辐射中包含的信息（比如文本、声音、图像等），这种攻击方式除了用于密码学攻击以外也被用于非密码学攻击等窃听行为，如TEMPEST攻击（例如范·埃克窃听、辐射监测）；
- 声学密码分析（Acoustic cryptanalysis ）：通过捕捉设备在运算时泄漏的声学信号捉取信息（与功率分析类似）；
- 差别错误分析（Differential fault analysis）：隐密资料在程序运行发生错误并输出错误信息时被发现；
- 数据残留（Data remanence ）：可使理应被删除的敏感资料被读取出来（例如冷启动攻击）；
- 软件初始化错误攻击（Software-initiated fault attacks）：现时较为少见，行锤（Row Hammer）攻击是该类攻击方式的一个实例，在这种攻击实现中，被禁止访问的存储器位置旁边的存储器空间如果被频繁访问将会有状态保留丢失的风险；
- 光学方式（Optical）：即隐密资料被一些视觉光学仪器（如高清晰度相机、高清晰度摄影机等设备）捕捉。

所有的攻击类型都利用了加密/解密系统在进行加密/解密操作时算法逻辑没有被发现缺陷，但是通过物理效应提供了有用的额外信息（这也是称为“旁路”的缘由），而这些物理信息往往包含了密钥、密码、密文等隐密资料。

---

## 参考文献

- [iOS 取证技术 - elcomsoft blog](https://blog.elcomsoft.com/2023/03/perfect-acquisition-part-1-introduction/)
- [揭开 Apple Touch ID 的神秘面纱](https://medium.com/hackernoon/demystifying-apples-touch-id-4883d5121b77)
- [一场椭圆曲线的寻根问祖之旅](https://fisco-bcos-documentation.readthedocs.io/zh-cn/v2.8.0/docs/articles/3_features/36_cryptographic/elliptic_curve.html)
- [椭圆曲线密码学 - Wiki](https://en.wikipedia.org/wiki/Elliptic-curve_cryptography)
- [ANSI X9.63 Overview](https://csrc.nist.gov/CSRC/media/Events/Key-Management-Workshop-2000/documents/x963_overview.pdf)
- [密钥管理的国际标准汇总](https://www.antpedia.com/standard/sp/65347.html)

### 官方文档下载

- [Apple 平台安全保护 - 2022年中文版](apple-platform-security-guide-cn-2022.pdf)
- [Apple SKS 加密模块（FIPS 140-2 非专用设备安全策略）- v10注释版](Apple_Secure_Key_Store_Cryptographic_Module_with_Notes.pdf)
- [TOE 评估报告 - MacOS 13：FileVault](Apple_MacOS_13_Ventura_FileVault_Security_Target.pdf)
- [TOE 评估报告 - iOS 16](Apple_iOS_16_iPhone_Security_Target_v1.1.pdf)
- [Apple T2 安全芯片概览](Apple_T2_Security_Chip_Overview_2018.pdf)
- [Behind the Scenes with iOS Security - Ivan Krstić](Behind_the_Scenes_with_iOS_Security.pdf)
- [Behind the Scenes with iOS Security 演讲视频](https://www.youtube.com/watch?v=BLGFriOKz6U)

### 研究报告下载

- [iOS Platform Security](Platform_Security.pdf)
- [Data Security on Mobile Devices](Data_Security_on_Mobile_Devices.pdf)
- [Demystifying the Secure Enclave Processor](us-16-Mandt-Demystifying-The-Secure-Enclave-Processor.pdf)
