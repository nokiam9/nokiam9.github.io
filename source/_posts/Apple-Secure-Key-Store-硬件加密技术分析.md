---
title: Apple Secure Key Store 硬件加密技术分析
date: 2024-04-07 21:08:41
tags:
---

根据[iOS 16 安全评估报告](Apple_iOS_16_iPhone_Security_Target_v1.1.pdf)的表述（P16-P20），Apple 设备提供的加密服务分为三个层级：

- User Space：SL1，软件级。驻留在用户空间内的动态可加载库，为 App 提供加密功能
- Kernel Space：SL1，软件级。操作系统内核加载的扩展模块，仅为内核提供加密功能
- Secure Key Store：SL2，硬件级。一个单芯片独立硬件加密模块，为静态数据和数据传输提供保护，包含固件和硬件加密算法实现

Secure Key Store（安全密钥存储，SKS，也称 Secure Key Service）是 Apple Secure Enclave 的加密模块，也是其硬件加密技术的核心组件。

- 根据 Apple 官网提供的[第三方平台认证信息](https://support.apple.com/zh-cn/guide/certifications/welcome/web)，其硬件、操作系统、应用软件、互联网服务和 Apple Pay 广泛采用 FIPS 140-2 加密标准（FIPS 140-3 认证已提交但尚未通过），当前[硬件加密模块 v10.0](https://csrc.nist.gov/CSRC/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp3811.pdf) 获得 CMVP 颁发的[3811 证书](https://csrc.nist.gov/Projects/cryptographic-module-validation-program/Certificate/3811)。
- Apple 积极参与通用标准 (CC) 评估，当前[认证产品清单](https://www.niap-ccevs.org/Product/index.cfm)包括：macOS 13 Ventura（含[FileVault](https://www.niap-ccevs.org/MMO/Product/st_vid11348-st.pdf)）、iOS 15 & [iOS 16](https://www.niap-ccevs.org/MMO/Product/st_vid11349-st.pdf)；iPadOS 15 & iPadOS 16，由 [ATSEC 公司](https://www.atsec.com)出具 TOE（Target of Evaluation）评估报告。
- 2016年美国举办的 Black hat 会议上，Apple 公司的 Ivan Krstic 介绍了数据保护技术，也是难得的[官方资料]((Behind_the_Scenes_with_iOS_Security.pdf))。

本文就是依据上述公开信息的研究分析。

## 一、Secure Key Store 总体架构

### 1. 硬件环境

根据[Apple SKS 加密模块](Apple_Secure_Key_Store_Cryptographic_Module_with_Notes.pdf)的表述，其逻辑边界包含以下组件：

![SKS](SKS-arch.png)

- SEP（Secure Encalve Processor）：区别于 REE 环境的应用处理器 AP，位于 TEE 环境的安全隔区拥有独立的应用处理器
    SEP 运行的操作系统称为 sepOS（基于 L4 微内核的 Apple 定制版本）
    TOE OS 和 sepOS 之间采用物理隔离方式，仅能通过 Mailbox 完成数据交互
- SKS（Secure Key Store，安全密钥存储）：负责生成并管理 CSP（Critical Security Parameter，关键安全参数），为静态数据加密所需密钥提供关键的加密材料
    按照 FIPS 140-2 的要求提供 POST（Power-On Self Test，启动自检）功能，以确保模块的完整性和加密功能的正确性
- HW AES：基于硬件实现的 AES 加密算法，Apple 安全隔区支持对抗时序攻击、静态功耗分析（SPA）和动态功耗分析 (DPA)。
- UID：在 SoC 生产环节，使用基于 AES 算法的 DRBG 生成 UID，然后蚀刻到硅片上的 OTP-ROM（One Time Programmable Read-Only Memory，一次性编程的只读内存，即熔丝），它只能通过 AES 引擎处理加密和解密操作，即使 SEP 的 Apps 也无法读取
- HW DRBG：基于硬件实现的伪随机数发生器（确定性随机数发生器），内置专用的 AES 引擎实现 CTR_DRBG 算法

> 早期的 A8 处理器的硬件版本是 1.0，A9 及后续处理器升级为 2.0，差异在于是否提供 PAA (Processor Algorithm Accelerator，处理器算法加速，基于 ARM NEON 指令集实现)。

对比 Apple 平台安全技术白皮书提供的信息，HW AES = Secure Enclave AES Engine，此外还有 PKA（public key acceleration，公钥加速器）。

![SoC](soc.png)

### 2. 应用架构

根据[Demystifying the Secure Enclave Processor](us-16-Mandt-Demystifying-The-Secure-Enclave-Processor.pdf)的 P61 的描述，Secure Key Store 作为 SEPOS 的一个功能模块，其暴露在 SL1 级（内核 XNU）的 Endpoint 就是一个内核扩展`AppleKeyStore.kext`。此外，SEPOS 还包含了其他类型的 APP，包括：

- ART Manager：反重放攻击服务
- Secure Biometric Engine：生物识别引擎，简称 SBIO，用于 Touce ID 和 Face ID 等
- Secure Credential Manager：凭据管理，用于网络连接和配件的安全管理
- SSE：未知。似乎用于硬件识别？

![SEPOS](sepos.png)

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

> 注意，系统密钥包**不存储 Class D key**！因为 NSFileProtectionNone 实际是一个单密钥系统，即：`per-file key = Class D key = AES_Unwrap(key 0x835，Dkey)`

### 1. 关键安全参数

[iOS 16 安全评估报告](Apple_iOS_16_iPhone_Security_Target_v1.1.pdf)的 P119 - P121，介绍了密钥管理的基本概念并提供了 CSP（Critical Security Parameter，关键安全参数）的列表信息，P125 提供了密钥层次结构图，与 iOS4 的描述基本一致，

![TOC](<TOC-key.png>)

[Apple SKS 加密模块](Apple_Secure_Key_Store_Cryptographic_Module_with_Notes.pdf)的 P20，也提供了关键安全参数 CSP 的生命周期列表：

- `Root Encryption Key (REK)` = Passcode Key，其描述为：
    Generation：Derived from passcode using PBKDF2 and entanglement with UID
    Entry and Output：Generated inside the module and is never output。
- `Moudle-managed keybags key` **不是 Bag1**(因为其是明文存储方式)，而是指密钥包中存储的各种 Class key
    Generation: Symmetric key generation services of the module
    Entered encrypted using AES-256 KW, Output encrypted using AES-256 KW
    Non-volatile store: cryptographically zeroized when overwriting the KEK
    Volatile store: zeroized when freeing the secure memory
- `File system object DEK` = EMF
- `Escrow Keybag and the class D key encryption key` = Key 0x835
- CSP 列表没有提供 Key 0x89B 的信息

> Q1： CSP 列表的 `REK wrapping key` 应该就是 SKS 和 SBIO 之间的 random secret。
> 描述为：Entered/Output in plaintext by calling application within physical the boundary。
> random secret 在 SKS 和 SBIO 之间明文传输，但都处于 SEP 范围内，行为方式符合其描述

### 2. Effaceable Storage

根据[iPhone Data Protection in Depth](iPhone_Data_Protection_in_Depth.pdf)的介绍，从 iPhone 3GS 开始，iPhone 内置的闪存芯片都最前面的 16 个 block 和最后的 15 个 block 保留给 Apple，并将 Block 1 作为 Effaceable Storage（可擦除区域），用于储存加密密钥的专用区域，可被直接寻址和安全擦除。

![effaceable storage](ES.png)

Github 上有一个取证软件包可以读取早期的 iOS 系统数据，请参见[https://github.com/nabla-c0d3/iphone-dataprotection](https://github.com/nabla-c0d3/iphone-dataprotection)。

浅蓝色是 length ，红色是 tag 标记，注意 HFS 是大端字节顺序，字符串要反着读。
第一个标记：0x42414731 = BAG1，长度 0x0034 = 52 个字节
第二个标记：0xC46B6579 = key，长度 0x0028 = 40 个字节
第三个标记：0xC54D4621 = EMF!，长度 0x0024 = 36 个字节
第四个标记：0x444F4E45 = DONE，长度 0x0000，这就是结束了！

![effaceable storage](ES2.png)

尽管当攻击者实际占有设备时，可擦除存储器无法提供保护（事实上 OS 可以直接读取该区域），但其中存储的密钥作为密钥层级的一部分，可以实现快速擦除和前向安全性。

![effaceable storage](ES3.png)

### 3. 密钥包

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
- HMCK：如果密钥包已签名，保存 HMAC 校验值
- WRAP：包裹方式，[1:仅 UID 派生, 2:仅 passcode 派生, 3:两者都有]
- SALT：PBKDF2 算法使用的盐
- ITER：PBKDF2 算法的迭代次数

> 不同版本可能出现 GRCE、GRNT、TKMT、SART 等字段，信息含义未知

对于每个密钥包的条目（文件保护类密钥 & 钥匙串类密钥），其字段包括：

- CLAS：class，密钥的级别
    文件保护类型为[1,2,3]：4-空缺，就是Dkey；5-保留未启用；
    钥匙串保护类型为[6,7,8,9,10,11]：对应钥匙串的 3 个级别和相应的 device only
- WRAP：wraping type，包裹方式。与 Header 定义一致
- KTPY：key type。例如 Curve25519
- WPKY：wrapped key。如果是非对称密钥，此处存储的是 Private key
- PBKY：public key（可选）。Class B 的 Pulic key，包裹状态
- UUID：该密钥的 UUID

关于密钥包的解封流程，[Apple SKS 加密模块](Apple_Secure_Key_Store_Cryptographic_Module_with_Notes.pdf)的 P23 指出：

- 所有在钥匙包中、由加密模块管理的密钥，由 Device OS 负责存储在 non-volatile memory 中。
- 密钥包基于 256 位的 AES-KW 包裹算法，然后输出到 Device OS 完成持久化存储（permanent storage）。
- 设备加电启动后，加密模块从 Device OS 导入包裹状态的密钥包并完成解封。

P14 描述了基于密码的身份验证工作模式：

- 当用户从模块请求加密服务时，它必须提供密码和对用户密钥包的引用，该密钥包在 SKS 内的 SP800-38F AES 密钥包装 (AES-KW) 下加密存储。该模块使用 PBKDF 从操作者提供的密码中派生 AES 密钥。
- 然后，模块的 SP800-38F AES 密钥解包功能（即 AES-KW-AD）使用派生的 AES 密钥来解密参考用户密钥包并验证解密密钥的真实性。
    由于 AES-KW 是一种身份验证密码，因此解密操作只有在没有身份验证错误的情况下才会成功。这意味着用户提供了正确的密码来导出用于 AES 密钥解包的正确 AES 密钥。
    任何其他密码都将派生出不同的 AES 密钥，这将导致解密的用户密钥错误，导致身份验证检查失败。
- 如果可以成功解开用户密钥包，则用户将通过模块的身份验证，然后将使用解开的用户密钥继续执行所请求的加密服务。
    解包用户密钥包失败也是用户身份验证失败，操作员将被拒绝访问模块。

## 四、Secure Key Store 业务逻辑

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

上图是一个简要描述，[iOS 16 安全评估报告](Apple_iOS_16_iPhone_Security_Target_v1.1.pdf)的 P126 提供了更详细的步骤，但似乎存在一些偏差，后续有勘误分析！

- XNU 首先提取文件元数据（使用 EMF 密钥加密），并将其发送到 SEP
- SEP 持有 EMF 密钥，据此解密文件元数据，并将其发送回 XNU
- XNU 判断文件保护类型，并将文件密钥（用类密钥包裹）和**类密钥标记**发送到 SEP
- SEP 解开文件密钥，使用临时密钥重新包裹，并将其返回 XNU
- XNU 将文件访问请求（读/写）与重新包裹的文件密钥一起发送到存储控制器。
- 存储控制器使用其内部的 AES 硬件解密文件密钥，然后在从闪存传输数据或向闪存传输数据期间解密（读取操作时）或加密（写入操作时）数据

## 五、 Secure Key Store 服务接口

[Apple SKS 加密模块](Apple_Secure_Key_Store_Cryptographic_Module_with_Notes.pdf)的 P15-P17，提供了 SKS 系统服务清单。

### 1. 完全符合 FIPS 140-2 的系统服务

1. 文件系统服务（Class D File System Services）：无保护类型文件的系统（读/写）服务
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

## 六、MacOS FileVault 的技术分析

与 iPhone OS 相比，MacOS 的加密技术方案 FileVault 有着明显差异，一是仅支持单密钥的 FDE 加密方案，而非数据保护技术的 FBE 加密方案；二是 Macbook 长期使用 Intel CPU，硬件加密只能采用 T2 安全芯片的外挂方案。

NIAP 根据 CC 安全标准定义了 FDE 的保护配置文件，请参见[https://www.niap-ccevs.org/Profile/PP.cfm](https://www.niap-ccevs.org/Profile/PP.cfm)，其核心功能组件分别为 AA（Authorization Acquisition，授权获取）和 EE（Encryption Engine，加密引擎），可以由不同的供应商提供组合能力。
![FDE](FDE1.png)

一般来说，AA 和 EE 的运行环境可能会根据其运行平台的启动阶段而有所不同，初始化方面以及可能的授权可以在 Pre-Boot 中执行，同时配置、加密、解密和管理功能可能在 OS 中执行。

![FDE](FDE2.png)

AA 组件有三种类型的加密因子，分别是设备厂商提供的硬件密钥，用户提供的 passcode，和 TRNG 随机生成的 salt，共同构成了机密性保护。
根据 [TOE 评估报告 - MacOS 13：FileVault](Apple_MacOS_13_Ventura_FileVault_Security_Target.pdf)，P9 指出了 Apple 自研 CPU 负责实现 AA 和 EE 的全部功能，而 Intel CPU 只负责 AA 的 Password Acquisition 功能的实例化，其他核心功能仍然由 T2 安全芯片提供，具体对比见下图。

![Intel](filevault2.png)
![M1](filevault1.png)

## 七、几个问题的讨论

### 1. 安全存储组件的数据保存在哪里？

根据[Apple平台安全保护-2022](https://help.apple.com/pdf/security/zh_CN/apple-platform-security-guide-cn.pdf)的表述：

- 虽然安全隔区不含储存设备，但它拥有一套将信息安全储存在所连接储存设备上的机制，**该储存设备与应用程序处理器和操作系统使用的 NAND 闪存互相独立**。
- 安全隔区配备了**专用的安全非易失性存储器设备**。安全非易失性存储器通过专用的 I2C 总线与安全隔区连接，因此它仅可被安全隔区访问。所有用户数据加密密钥植根于储存在安全隔区非易失性存储器中的熵内。
- 在未配备安全储存组件的架构上，**EEPROM（电可擦除可编程只读存储器）被用于为安全隔区提供安全储存服务**。跟安全储存组件类似，EEPROM 连接到安全隔区并仅可从安全隔区访问，但其不包含专用的硬件安全性功能，不能确保对熵的独有访问权限（除了其物理连接特性），也不具备计数器加密箱功能。

总结一下，安全存储组件（Secure Storage Component）的（持久化）存储设备是一个专用芯片（早期是 EEPROM，但**绝对不是 NAND 的 Effaceable Storage**），通过专用 I2C 总线与安全隔区连接，即 Secure Encalve 标注的**安全非易失性存储器（Secure nonvolatile storage）**！

### 2. 安全存储组件管理了哪些核心数据？

根据[Apple平台安全保护-2022](https://help.apple.com/pdf/security/zh_CN/apple-platform-security-guide-cn.pdf)的表述：

- 搭载 A12、 S4 及后续型号 SoC 的设备并用了安全隔区与**安全储存组件来储存熵**。安全储存组件本身设计为使用不可更改的 ROM 代码、硬件随机数生成器、**每个设备唯一的加密密钥**、加密引擎和物理篡改检测。安全隔区和安全储存组件使用加密且认证的协议通信以提供对熵的独有访问权限。
- 2020 年秋季或之后首次发布的设备（为抵御重放攻击）配备了第二代安全储存组件。第二代安全储存组件增加了计数器加密箱（和 xART 机制）。**每个计数器加密箱储存一个 128 位盐、一个 128 位密码验证器、一个 8 位计数器，以及一个 8 位最大尝试值**。对计数器加密箱的访问通过加密且认证的协议来实现。
- 计数器加密箱中含有所需用于解锁受密码保护用户数据的熵。若要访问用户数据，**配对的安全隔区必须从用户的密码和安全隔区的 UID 中派生出正确的密码熵值**。从除配对安全隔区之外其他来源发送的解锁尝试均无法获知用户的密码。如果密码的尝试次数超过限制（例如，在 iPhone 上为 10 次），安全储存组件就会完全抹掉受密码保护的数据。
- 为了创建计数器加密箱，安全隔区会向安全储存组件发送密码熵值和最大尝试次数值。安全储存组件会使用其随机数生成器生成盐值。之后**通过提供的密码熵、安全储存组件的唯一加密密钥和盐值派生出密码验证器值和加密箱熵值**。安全储存组件使用计数 0、提供的最大尝试次数值、派生的密码验证器值和盐值来初始化计数器加密箱。之后**安全储存组件将生成的加密箱熵值返回到安全隔区**。
- 对于搭载 A9 或后续型号 SoC 的设备，该 .plist 文件（密钥包）包含一个密钥，表示密钥包储存在受**反重放随机数（由安全隔区控制）**保护的有锁储存库中。

做个初步的分析。

1. 密码熵值和加密箱熵值在安全隔区和安全存储之间传递，这是一个标准的双向鉴权流程。
    密码熵值可以肯定就是 Passcode Key，加密箱熵值估计是系统密钥包的 SART 字段，即安全白皮书介绍的 A9 设备的新增密钥。
    早期版本的 HMCK 字段保存签名信息，但目标是密钥包内容的完整性，而非反重放攻击。
2. 核心密钥全部由 Secure Key Store 管理，安全储存组件仅负责存储一些关键的安全参数，包括：反重放随机数（区别于系统密钥包中、用于 REK 的 Salt）、密码验证器、计数器和最大尝试值等，这些数据不以任何形式输出，为 xART 机制提供额外的机密性。
3. 安全储存组件的唯一加密密钥（即 xART key）的构造方式不清楚，猜测是一个 UID 的固定衍生密钥（如果是原生密钥，就需要额外考虑持久化存储方式）。

### 3. 操作系统绑定密钥的工作机制？

SKP（Sealed Key Protection，密封密钥保护，也称操作系统绑定密钥），是使用系统软件的测量值和仅在硬件中可用的密钥来保护（或密封）加密密钥的一种技术。搭载 Apple 芯片的 Mac 上，对 KEK 的保护通过整合有关系统安全性策略的信息进一步得到了加强。

![SKP](SKP.png)

- Apple 设备支持一项称为密封密钥保护 (SKP) 的技术，其旨在确保加密材料在这些情况下不可用：脱离设备时，或者对操作系统版本或安全性设置存在未经用户正确授权的操纵时。**此功能不是由安全隔区提供，而是由位于更底层的硬件寄存器支持**，目的是针对解密用户数据所需的密钥提供独立于安全隔区的额外保护层
- 由用户密码与长期 SKP 密钥和硬件密钥 1（安全隔区的 UID）配合使用而生成的密钥称为密码派生密钥。**此密钥用于保护用户密钥包（在所有支持的平台上）和 KEK（仅限在 macOS 中）**，然后启用生物识别解锁或使用其他设备（如Apple Watch）自动解锁
- 在搭载 Apple 芯片的 Mac 中，LLB（Low Level Bootloader，底层引导载入程序）会验证设备是否存在有效的 LocalPolicy，且 LocalPolicy 策略随机数值是否与**安全储存组件中所储存的值匹配**
- 系统还会使用一个独立于 lpnh 和 rpnh 的随机数，以便设备被“查找”置于停用状态时，现有操作系统可被停用（通过将其 **LPN 和 RPN 从安全储存组件中移除**）的同时仍保持系统 recoveryOS 可启动

![搭载 Apple 芯片的 Mac 开机时的启动过程步骤](mac-boot.png)

初步探讨一下：

1. Hardward key 1 = UID，Hardward key 2 = Key 0x835
2. LPN（Local Policy Number，本地策略随机数）和 RPN（Remote Policy Number，远程策略随机数）是用于 LLB 校验的策略随机数值，也在安全存储组件中保存。
3. passcode key 仍然由 passcode 和 UID 通过 PBKDF2 算法生成，但是增加了 LPN 和 RPN 的校验环节以支持 SKP 功能。
4. 操作系统绑定的目的是阻止用户安装未经 Apple 授权的操作系统，从根本上杜绝了 iOS 越狱功能。对于个人使用的 iPhone，其只能通过 Apple Store 安装官方认证的 APP，但是对于生产力工具 Macbook 来说，必须支持用户手工安装应用程序，因此设定了不同的安全等级来支持。

### 4.  `Key 0x89B` 和 `Key 0x835` 存储方式的勘误

[iOS 16 安全评估报告](Apple_iOS_16_iPhone_Security_Target_v1.1.pdf)的 P117 说明：
The UID is used to derive two other keys, called "Key 0x89B" and "Key 0x835". These two keys are derived during **the first boot** by encrypting defined constants with the UID.

P121 的密钥列表中，关于这两个 UID 衍生密钥的存储方式描述是：
SEP. Block 0 of the flash memory. (Effaceable storage.)

P125 进一步描述：
The UID (a.k.a. UID key) is not accessible by any software. The "Key 0x89B" and "Key 0x835" keys are both derived by encrypting **defined values (identical for all devices)** with the UID key. All three keys are stored in the SEP. All other keys shown in the figure are stored in wrapped form in persistent storage and unwrapped when needed.

笔者认为，这两个 UID 衍生密钥应该是安全隔区**每次**启动时，基于固定值计算得出并保存在内存中，无需持久化存储。

- Key 0x835 = AES_KW(UID, 0x’01010101010101010101010101010101’)：Dkey 包裹密钥
- Key 0x89B = AES_KW(UID, 0x’183e99676bb03c546fa468f51c0cbd49’)：EMF 包裹密钥
- Key 0x836 = AES_KW(UID, 0x‘00E5A0E6526FAE66C5C1C6D4F16D6180’)
- Key 0x837 = AES_KW(GID, 0x’345A2D6C5050D058780DA431F0710E15’)：Apple 固件的 img3 文件解密
- Key 0x838 = AES_KW(UID, 0x’8C8318A27D7F030717D2B8FC5514F8E1‘)

当然也有一种可能性，就是设备第一次启动时生成，并保存在安全存储组件的 Flash 中，这就留待后续验证吧。

### 5. 关于文件读写流程的勘误

[iOS 16 安全评估报告](Apple_iOS_16_iPhone_Security_Target_v1.1.pdf)的 P126 提供了详细的文件读写流程，但似乎存在一些偏差，其描述为：

> XNU 判断文件保护类型，并将类密钥（用 Dkey 包装，或 Dkey XOR Passcode Key 包装）和文件密钥（用类密钥包装）发送到 SEP

笔者的分析认为，内部的 SKS keyring 保存已解封的类密钥，尚未解封类密钥也保存在 SKS memory ，因此 XNU 无需发送 wrapped class key，仅提供 class key type 即可，请参见安全白皮书的描述：

> 当打开一个文件时，系统会使用文件系统密钥解密文件的元数据，以显露出**封装的文件独有密钥**和表示它受哪个**类保护的记号**。文件独有或范围独有密钥使用类密钥解封，然后提供给硬件 AES 引擎，该引擎会在从闪存中读取文件时对文件进行解密。所有封装文件密钥的处理发生在安全隔区中；文件密钥绝不会直接透露给应用程序处理器。启动时，安全隔区与 AES 引擎协商得到一个临时密钥。当安全隔区解开文件密钥时， 它们又通过该临时密钥再次封装，然后发送回应用程序处理器。

此外，iOS 4 破解技术中介绍 WARP 类型‘3’ 是两轮解封，硬件加密简化为：`Dkey XOR PDK`？

---

## 附录：Cocoa 框架

Cocoa 是苹果公司为 macOS 所创建的原生面向对象的应用程序接口，是 Mac OS X 上五大 AP 之一（其它四个是Carbon、POSIX、X11 和 Java）。

Cocoa 应用程序一般在苹果公司的开发工具 Xcode（前身为 Project Builder ）和 Interface Builder 上用Objective-C 写成。

Cocoa包含三个主要的 Objective-C 对象库，称为“框架”。框架的功能类似于动态库，即可以在运行时动态的加载应用程序的地址空间，但框架作为一个捆绑而非独立文件，其中除了可执行代码外，也包含了资源，头文件和文档。

- Foundation：面向对象的通用函数库。提供了字符串，数值的管理，容器及其枚举，分布式计算，事件循环，以及一些其它的与图形用户界面没有直接关系的功能，其中用于类和常数的函数有`NS`前缀（因为源自于NeXTSTEP）。
    它可以在 MacOS 和 iOS 中使用。
- AppKit（Application Kit，应用程序工具包）：包含了程序与图形用户界面交互所需的代码。基于 Foundation 创建的，也使用`NS`前缀（因为其直接派生自 NeXTSTEP）。
    它只能在 MacOS 中使用。
- UIKit（User Interface Kit，用户界面工具包）：用于 iOS 的图形用户界面工具包。与AppKit不同，它使用`UI`的前缀。

---

## 参考文献

- [iOS 取证技术 - elcomsoft blog](https://blog.elcomsoft.com/2023/03/perfect-acquisition-part-1-introduction/)
- [揭开 Apple Touch ID 的神秘面纱](https://medium.com/hackernoon/demystifying-apples-touch-id-4883d5121b77)
- [一场椭圆曲线的寻根问祖之旅](https://fisco-bcos-documentation.readthedocs.io/zh-cn/v2.8.0/docs/articles/3_features/36_cryptographic/elliptic_curve.html)
- [椭圆曲线密码学 - Wiki](https://en.wikipedia.org/wiki/Elliptic-curve_cryptography)
- [ANSI X9.63 Overview](https://csrc.nist.gov/CSRC/media/Events/Key-Management-Workshop-2000/documents/x963_overview.pdf)
- [密钥管理的国际标准汇总](https://www.antpedia.com/standard/sp/65347.html)
- [EEPROM 与 Flash 的区别与比较](https://askanydifference.com/zh-CN/difference-between-eeprom-and-flash-with-table/)

### 官方文档下载

- [Apple 平台安全保护 - 2022年中文版](apple-platform-security-guide-cn-2022.pdf)
- [Apple SKS 加密模块（FIPS 140-2 非专用设备安全策略）- v10注释版](Apple_Secure_Key_Store_Cryptographic_Module_with_Notes.pdf)
- [TOE 评估报告 - MacOS 13：FileVault](Apple_MacOS_13_Ventura_FileVault_Security_Target.pdf)
- [TOE 评估报告 - iOS 16](Apple_iOS_16_iPhone_Security_Target_v1.1.pdf)
- [Apple T2 安全芯片概览](Apple_T2_Security_Chip_Overview_2018.pdf)
- [Behind the Scenes with iOS Security - Ivan Krstić](Behind_the_Scenes_with_iOS_Security.pdf)
- [Behind the Scenes with iOS Security 演讲视频](https://www.youtube.com/watch?v=BLGFriOKz6U)

### 研究报告下载

- [iPhone Data Protection in Depth - Sogti](iPhone_Data_Protection_in_Depth.pdf)
- [iOS Encryption - NCC Group](2016-BSidesROC-iOSCrypto.pdf)
- [iOS Platform Security](Platform_Security.pdf)
- [Data Security on Mobile Devices](Data_Security_on_Mobile_Devices.pdf)
- [Demystifying the Secure Enclave Processor](us-16-Mandt-Demystifying-The-Secure-Enclave-Processor.pdf)
- [iOS Encryption Systems - 奥地利格拉茨技术大学](iOS_Encryption_Systems.pdf)
- [iOS 5的数据保护技术分析](0721C6_Andrey.Belenko_Evolution.of.iOS.Data.Protection.pdf)
- [iPhone 数据保护技术 - ELComSoft](OWASP_BeNeLux_Day_2011_-_A._Belenko_-_Overcoming_iOS_Data_Protection.pdf)
