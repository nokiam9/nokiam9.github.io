---
title: Apple安全隔区（Secure Encalve）技术概览
date: 2022-10-03 13:01:50
tags:
---

## 一、Apple平台安全性 = 软件 + 硬件 + 服务

[apple平台安全白皮书](apple平台安全白皮书-2021中文版.pdf)指出，Apple平台安全性的核心理念是：强调硬件、软件和服务的紧密联系和共同协作，为智能终端（iOS）、台式设备（MacOS）、可穿戴设备（WatchOS）和家居设备等不同类型设备提供具备独特性的安全性架构，目标是在为用户提供最高的安全性和透明的使用体验，其中：

- 硬件：遵循“**支持有限且单独定义的功能**”的准则，依托Apple自行设计的芯片和安全性硬件，为关键的安全性功能提供支持，以使攻击面最小化
- 软件：通过软件保护为操作系统和第三方App提供了安全保障
- 服务：通过服务提供安全且及时的软件更新机制，帮助建立受保护的App生态系统，并保障通信和支付的安全

![技术架构](framework.png)

需要注意的是，Apple安全体系中有三个最核心的密钥，分别是：

- UID：一个 256 位的 AES 密钥，在设备制造过程中刻录在每个处理器上。这种密钥无法由固件或软件读取，只能由处理器的硬件 AES 引擎使用。UID 与设备上的任何其他标识符均无关，包括但不限于 UDID
- GID：设备组ID，类似于 UID，但同一类中的每个处理器的 GID 都相同
- Passcode & Password：用户密码，iOS中称为passcode，MacOS中称为password；由用户自行设置,一般是4位数字、6位数字或无限长度的字母组合

## 二、Secure Enclave的发展历程

对苹果来说，自研芯片已经成为了一种寻求产品差异化的手段，这不仅在 iPhone 上有所体现，你在 Mac 电脑、Apple Watch 手表和 AirPods 耳机中都能找到苹果芯片的身影。

### A系列主处理器（iOS）

2013年，Apple发布了iPhone 5S，首次支持了指纹识别设备Touch ID，为此需要解决一个关键问题，如何安全地存储生物特征数据？显而易见的是，单纯的软件方案或云端方案肯定无法获得消费者和监管机构的安全认可。
iPhone 5S的 CPU 是基于 ARMv8 架构的**A7**自研芯片，为了实现本地级别的加密和安全，其设计了一个专门的协处理器用于实现本地级别的加密和安全，这就是安全隔区Secure Enclave。
![A](apple-A.jpeg)

### T系列安全芯片（MacOS）

传统上，Mac个人电脑是基于Intel的芯片，因此在硬件安全性上受到严重的制约，为此Apple只能通过引入额外的 T2 芯片来支撑安全特性。

2016年，苹果开始为新 MacBook 加入名为**T1**的自研芯片，当时主要是为了驱动 TouchBar 触控条的运行，同时也负责指纹识别的安全管理。
2017年，苹果在发布 iMac Pro时，发布了升级版的 T2 芯片.
![T2](t2.png)

2020年，Apple发布了首款专为MacOS打造的M1芯片，将中央处理器、输入输出、安全隔区等功能统统整合在同一块芯片，为全面淘汰Intel CPU做好了准备，也表明T系列安全芯片即将终结。

### S系列主处理器（WatchOS）

2015年，Apple发布了Apple Watch，采用自研的S1芯片，运行watchOS操作系统，目前已经发展到 S8 和 WatchOS 9.0。
S系列处理器同样包含安全隔区，为生物特征数据的采集和存储提供安全解决方案。
此外，Apple TV HD 、HomePod 和 HomePod mini等各类Apple产品都搭载安全隔区。

总结一下，通过Secure Enclave技术，Apple实现了智能终端、生产力设备和可穿戴设备等各个平台的统一安全解决方案，将硬件和软件的整合提升到了一个全新的高度。

## 三、Secure Enclave的核心组件

一般认为，Secure Enclave就是一个高度定制版的 ARM TrustZone，有自己的安全引导过程和个性化的软件更新过程，并且同Rich OS所在的应用处理器分离，也就是“安全世界”的物理实现。

![SOC](soc.png)

安全隔区是一个独立于操作系统的片上系统（System on Chips），自身拥有独立的Boot ROM和AES引擎，为数据保护密钥管理提供所有加密操作，并保持数据保护的完整性（即使内核已被破坏）。
安全隔区还为加密静态数据所需密钥的安全生成和储存提供了基础，通过特殊通道向 AES 引擎提供所需的密钥材料，而不将此信息透露给应用程序处理器 (或 CPU) 或者整个操作系统。

### 1、安全隔区处理器（Secure Enclage Processor，SEP）

- 安全隔区处理器为安全隔区提供专用计算能力
- 运行Apple定制的L4微内核（也称为sepOS），包含了Apple的签名证书

### 2、内存保护引擎（Memory Protection Engine）

- 安全隔区使用宿主设备的内存硬件资源，但规划了专用区域，通过多层保护将由安全隔区保护的内存与应用程序处理器隔离
- 当设备启动时，安全隔区 Boot ROM 将生成一组随机临时密钥，用于内存保护引擎的加密处理和认证处理，并支持安全隔区内存的重放保护
- 内存保护引擎以内联方式运行且对安全隔区透明，即：安全隔区将内存当作普通的未加密内存一样进行读取和写入，而安全隔区外的观察程序只能看到加密和认证版本的内存
- 在 A14 和 M1 之后增加了第二组密钥，用于与安全神经网络引擎共享的数据

### 3、真随机数生成器（True Random Number Generator，TRNG）

- 用于生成安全的随机数，每次生成随机加密密钥、随机密钥种子或其他熵源时都会使用TRNG
- 基于多个并联环形振荡器的异或计算产生真随机数，再通过CTR_DRBG后处理算法提供安全的随机数

### 4、安全隔区AES引擎（Secure Enclage AES Engine）

- 安全隔区AES引擎是一个硬件块，用于基于 AES 密码来执行对称加密
- AES 引擎设计用于防范通过时序和静态功耗分析 (SPA) 泄露信息，还包括动态功耗分析 (DPA) 对策
- AES引擎支持软件密钥和硬件密钥，其内部保存了基于UID派生的硬件密钥，虽然sepOS可以请求通过硬件密钥进行加密和解密操作，但不能提取密匙

### 5、公钥加速器（Public Key Accelerator，PKA）

- 公钥加速器 (PKA) 是一个用于执行非对称加密操作的硬件块，支持 RSA 和 ECC (椭圆曲线加密) 签名和加密算法
- PKA 设计用于防范使用时序攻击及静态功耗分析（SPA）和动态功耗分析（DPA）等旁路攻击来泄露信息。
- PKA 支持软件密钥和硬件密钥，其内部保存了基于UID派生的硬件密钥，即使对sepOS软件也不可见
- 在Apple A10之后，PKA还支持操作系统绑定密钥，也称为密封密钥保护 (SKP)，其基于安全隔区Boot ROM的哈希值 + 设备UID的组合生成，由安全隔区启动监视器负责在请求特定 Apple 服务时验证 sepOS 版本，目的是在系统发生重大版本更改时，防止未授权用户的访问
  
## 四、Secure Enclave的附加组件

上述5个核心组件均属于计算资源范畴，与 ARM TrustZone 通用架构基本一致，但Apple安全隔区还有一些特殊类型的组件。

### 1. 安全隔区启动代码（Secure Enclave Boot ROM）

Apple 安全启动的设计目的是：仅允许来自 Apple 的受信任操作系统软件在启动时加载，并确保可保护软件的最底层不被篡改。
Boot ROM 是处理器在首次启动时所执行的第一个代码，一般固化在硬件中作为处理器不可分割的一部分，即使Apple也无法修改。
![boot rom](bootrom.png)

与主CPU的 Boot ROM 类似，安全隔区也包括一个专用的安全隔区 Boot ROM，是一段不可更改的代码，作为安全隔区的硬件信任根。
安全隔区启动的基本步骤包括：

1. 系统启动时，iBoot 会给安全隔区分配一个专用的内存区域；
2. 安全隔区 Boot ROM 作为安全隔区的硬件信任根，其首先初始化内存保护引擎，为由安全隔区保护的内存提供加密保护;
3. 应用程序处理器将 sepOS 镜像发送给安全隔区 Boot ROM，并拷贝到由安全隔区保护的内存；
4. 安全隔区 Boot ROM 检查 sepOS 映像的加密哈希值和签名，如果已正确签名以在设备上运行，将控制转移给 sepOS；否则，阻止进一步使用安全隔区直到下一次芯片被还原。
5. 在 A10 之后，安全隔区 Boot ROM 还将 sepOS 的哈希值锁定到专用寄存器，以便公钥加速器PKA用于操作系统绑定功能。

### 2. 安全隔区启动监视器（Secure Enclave Boot Monitor）

- 在 A13 之后，安全隔区还包括一个启动监视器，目标是阻止安全隔区处理器执行安全隔区 Boot ROM 以外的任何其他代码，以确保所启动 sepOS 的的完整性
- 启动监视器依赖安全隔区处理器的系统协处理器完整性保护 (SCIP) 配置实现保护功能
- 安全隔区 Boot ROM 启动时，通过检测 SepOS 镜像的哈希值以确保合法授权，而启动监视器负责确定并监控 sepOS 加载后的运行哈希值，以确保运行过程中代码不被外部篡改

### 3. 根加密密钥（Root Cryptographic Keys）

- 安全隔区包括一个唯一 ID (UID) 根加密密钥。UID 对于每台设备来说都是唯一的，且与设备上的任何其他标 识符都无关
- UID 是在制造过程中由安全隔区 TRNG 生成的随机数，并使用完全在安全隔区中运行的软件进程写入到**熔丝**中，Apple 或其任何供应商均无法访问或储存
- sepOS 使用 UID 作为加密因子生成各类密钥，从而将数据与特定设备捆绑起来。 例如，由于文件系统密钥层级中包括 UID， 因此如果将内置SSD储存设备从一台设备整个移至另一台设备，文件将不可访问，但通过USB连接的外部存储设备由于没有连接到AES 引擎，所以无法实现迁移保护；触控 ID 或面容 ID 数据的密钥也在受保护的范围内
- 安全隔区还有设备组 ID (GID)，它是使用特定 SoC 的所有设备共用的 ID (例如，所有使用 Apple A14 SoC 的设备共用同一个 GID)
- UID 和 GID 不可以通过联合测试行动小组 (JTAG) 或其他调试接口使用

> 熔丝，一般称为E-Fuse或OTP Fuse，本质就是内嵌的一块One Time Programmable Memory, 仅可被烧写一次，但可以被多次读取。

### 4、安全非易失性存储（Secure Nonvolatile Storage）

- 安全隔区配备了**专用**的安全非易失性存储器设备（区别于应用处理器的 NAND 存储设备），通过专用的 I2C 总线与安全隔区连接，确保物理层面仅可被安全隔区访问
- 早期使用 EEPROM 作为数据存储资源，从 A12、S4 开始采用了第一代安全存储组件，它与安全隔区之间使用加密且认证的通信协议，确保对数据资源的独有访问权限
- 2020年10月，发布了第二代安全组件，主要特性是新增了计数器加密箱，用于保护简单而易受攻击的用户密码passcode
- 安全储存组件设计用于以下关键业务场景的数据保护，包括：不可更改的 ROM 代码、硬件随机数生成器、每个设备唯一的加密密钥、加密引擎和物理篡改检测等

> 虽然Apple官方文档没有描述，但本人认为，安全存储组件可能存储的数据包括：
>
>- 生成密钥：通过UID、passcode和随机数等加密因子生成的各类密钥，需要持久化保存在安全载体之中
>- 生物特征：指纹识别和人脸识别的特征数据必须且只能保存在本机设备的安全载体之中
>- 各类计数器：保存状态数据，阻断重放攻击等反复尝试

### 5. AES引擎（AES Engine）

- 在安全隔区 AES 引擎之外，还有一个位于安全隔区边界外的 AES256 加密引擎，专门用于处理安全隔区和NAND (非易失性) 闪存之间的加密数据传输
- 在 A9 之后，闪存子系统位于隔离的总线上，该总线被授权只能通过 DMA 加密引擎访问包含用户数据的内存，因此必须为安全隔区设置该专用 AES 加密引擎
- 启动时，sepOS 会使用 TRNG 生成一个临时封装密钥，安全隔区使用**专用线路**将此密钥传输到 AES 引擎，旨在防止它被安全隔区外的任何软件访问；随后， sepOS 使用临时封装密钥来封装文件密钥，供应用程序处理器文件系统驱动程序使用，当文件系统驱动程序读取或写入文件时，它会将封装的密钥发送到 AES 引擎 以解封密钥。 AES 引擎绝不会将未封装的密钥透露给软件。
![AES-Engine](aes-engine.png)

### 6. 神经网络引擎（Secure Neural Engine）

- 在配备 Face ID 的设备上，安全神经网络引擎将 2D 图像和深度图转化为用户脸部的数学表达式（不是原始图像，仅包含特征值）
- 从 A11 开始，安全神经网络引擎已集成到安全隔区中。安全神经网络引擎采用直接内存访问 (DMA) 以实现高性能。由 sepOS 内核控制的输入输出内存管理单元 (IOMMU) 将此直接访问的范围限制在经授权的内存区域。
- 从 A14 和 M1 开始，安全神经网络引擎在应用程序处理器的神经网络引擎中作为安全模式实现。 一个专用的硬件安全性控制器会在应用程序处理器和安全隔区的任务间切换，每次转换时神经网络引擎状态均会被重设以保持面容 ID 数据的安全。 一个专用的引擎会应用内存加密、 认证和访问控制。 同时，它使用单独的加密密钥和内存范围，以将安全神经网络引擎限制在经授权的内存区域。

> 也就是说，安全神经网络引擎改为虚拟化方案，不再是独立的协处理器哦

### 7. 功耗和时钟监视器（Power and clock monitors）

- 所有的电子设备都被设计为在一定的电压和频率包络内运行。如果在此包络外运行，电子设备可能会发生故障，然后安全性控制就可能被绕过
- 为了帮助确保电压和频率保持在安全的范围内，安全隔区中设计了监视电路
- 这些监视电路被设计为具有比安全隔区其余部分更大的运行包络。如果监视器检测到非法运行点，安全隔区中的时钟会自动停止，在下一次 SoC 重设前不会重新开始运行。

## 五、数据安全的密钥体系

![Key](key.png)
![Key](key2.png)

sepOS 使用 UID 来保护设备特定的密钥。 有了 UID， 就可以通过加密方式将数据与特定设备捆绑起来。 例如， 用于保护文件系统的密钥层级就包括 UID， 因此如果将内置 SSD 储存设备从一台设备整个移至另一台设备， 文件将不可访问。 其他受保护的设备特定密钥包括触控 ID 或面容 ID 数据。 在 Mac 上， 只有链接到AES 引擎的完全内置储存设备才会获得这个级别的加密。 例如， 通过 USB 连接的外置储存设备或者添加到2019 年款 Mac Pro 的基于 PCIe 的储存设备都无法获得这种方式的加密。


![aaa](key.png)

## 六、几个问题的讨论

### 1. Apple Secure Enclave 的技术实现是虚拟化方案，还是协处理器方案？

由于Mac电脑原来采用Intel芯片，因此Apple发展了T系列安全芯片解决安全问题，显然这是一个独立芯片的方案，而随着Apple自研的M系列CPU的出现，安全隔区已经被整合到CPU芯片之中。
而从iPhone手机的角度看，A系列CPU都是基于ARM指令集，虚拟化方案和协处理器方案都可以满足TrustZone架构的技术要求，从外部功能提供上无法区别，主要差别就是设计复杂度和成本问题。

从Apple提供的安全性文档中的描述推测，Secure Enclave 采用了协处理器方案，请看关于SCIP的描述。
> Coprocessor firmware handles many critical system tasks—for example, the Secure Enclave, the image sensor processor, and the motion coprocessor. Therefore its security is a key part of the security of the overall system. To prevent modification of coprocessor firmware, Apple uses a mechanism called System Coprocessor Integrity Protection (SCIP).

但是也有例外，安全神经网络引擎原本是安全隔区的一个组件，但从 A14 开始，调整为应用处理器中神经网络引擎的安全模式，也就是虚拟化方案。考虑到神经网络引擎的专用性，这应该是更有效率的选择。
> Starting with A14 and the M1 family, the Secure Neural Engine is implemented as a secure mode in the Application Processor’s Neural Engine. A dedicated hardware security controller switches between Application Processor and Secure Enclave tasks, resetting Neural Engine state on each transition to keep Face ID data secure. A dedicated engine applies memory encryption, authentication, and access control. At the same time, it uses a separate cryptographic key and memory range to limit the Secure Neural Engine to authorized memory regions.

### 2. 如何确保 Secure Enclave 外部接口的安全性？

Secure Enclave 虽然是安全世界，但要正常工作就必须与外部世界发生数据交换，因此外部接口的边界安全防护是重点，必须全面、可靠。

- RAM：共享，使用主处理器的内存硬件资源，但通过安全内存保护引擎实现加密和认证
- ROM：专用，安全隔区有自己的Boot ROM，并完全固化在内部
- NAND（闪存）：可访问，安全隔区可以访问系统NAND资源，但通过专用AES引擎实现加密保护
- 安全非易失性存储：专用，存储各类密钥、熵源、计数器和指纹特征等用户敏感数据，通过专用I2C总线连接
- UID & GID：制造过程中通过E-Fuse固化

---

## 附录一：Apple自研芯片的安全隔区特性

![List](list.png)

## 附录二：真随机数生成器，伪随机数发生器 和 熵源

随机数在安全技术中通常被用于生成随机序列，而要做到“完全随机”是非常困难的，一般需要满足3个条件：

1. 其必须是不可预知的
2. 其必须被均匀分布(即任何数均有平等出现的机会)
3. 其必须统计独立 (即与以前生成的随机数不存在明显关系)

美国国家标准和技术研究所(NIST)制定了随机数发生器的技术规范，称为`NIST SP 800 - 22`和`SP 800 - 90`，其推荐的技术路线是：真随机数生成器TRNG + 伪随机数生成器PRNG。

真随机数通常来源于硬件随机数生成器，每次生成的随机数都是真正的随机数，但是因为物理因素，生成的时间较慢。比如STM32中提供的RNG硬件外设。
随机数发生器(TRNG)从一些随机过程中收获熵(如:晶体管中电流产生的噪音，或放射性衰变事件之间的时间)，继而通过异或运算等方法消除随机位的概率偏差，最后输出高质量的随机数

伪随机数通常来源于某个生成算法，每次生成的随机数虽然是随机的，但还是遵循生成算法的规则，优点是生成速度较快。比如C库中提供的rand函数，在相同的种子下，其生成的随机数序列相同，所以称之为伪随机数。
PRNG（也叫做确定性随机数生成器，DRBG）的最佳输入设备就是高质量的TRNG，伪随机数所使用的种子就称为熵源（熵池）。

NIST SP 800-90A规范中描述了三种产生伪随机数的算法：

- Hash_DRBG：使用单向散列算法作为伪随机数生成的基础算法；
- HMAC_DRBG：使用消息认证码算法作为随机数生成的基础算法；
- CTR_DRBG：使用分组密码算法的计数器模式作为随机数生成的基础算法（重点）。

CTR-DRBG 可以理解为一个AES加密过程，通过利用分组密码算法对一个递增的计数器进行加密来产生期望随机数序列，分组密码算法可以是3DES(64bits)，AES(128, 192, 256bits)。

熵源
内核 CPRNG 源自启动过程中的多个熵源并存在于设备的整个生命周期。 这些来源包括 (取决于可用性):
• 安全隔区硬件 TRNG
• 启动过程中所收集基于时序的时间误差
• 从硬件中断收集的熵
• 用于启动过程中保持熵的种子文件
• Intel 随机指令， 例如 RDSEED 和 RDRAND (仅限基于 Intel 的 Mac)

内核 CPRNG
内核 CPRNG 的设计源自 Fortuna 算法， 旨在满足 256 位安全级别。 它使用以下 API 为用户空间使用者
提供高质量的随机数 :
• getentropy(2) 系统调用
• 随机设备 (/dev/random)
内核 CPRNG 通过写入随机设备接受用户提供的熵。

---

在 A11 和更新版本以及 S3 和更新版本的 SOC 上，完整性树用于防止对安全至关重要的 Secure Enclave 内存的重放，通过内存保护密钥和存储在片上 SRAM 中的随机数进行身份验证。
Secure Enclave 保存到文件系统的数据使用与 UID 纠缠的密钥和反重放计数器进行加密。防重放计数器存储在专用的非易失性存储器集成电路 (IC) 中。
在配备 A12 和 S4 SoC 的设备上，Secure Enclave 与用于防重放计数器存储的安全存储集成电路 (IC) 配对。安全存储 IC 设计有不可变 ROM 代码、硬件随机数生成器、加密引擎和物理篡改检测。为了读取和更新计数器，Secure Enclave 和存储 IC 采用了确保对计数器的独占访问的安全协议。

## 附录三：第二代安全存储组件的计数器加密箱的工作原理

2020年10月，苹果突然发布了第二代安全存储组件，并紧急升级 A12、A13 以及 S5 芯片，据认为 GrayKey 密码破解设备有关系，其采用暴力破解方式实现iPhone解锁。

第二代安全储存组件增加了计数器加密箱，包括：1个128位盐，1个128位密码验证器，1个8位计数器，1个 8 位最大尝试值，其工作原理如下：

### 1. 创建阶段

- 安全隔区：
    发送数据：密码熵值（passcodeEntropyValue），最大重试次数（maxRetryTimes）
    接受数据：密码箱熵值（lockboxEntropyValue）
- 安全存储组件：
    自有数据：安全存储组件的唯一加密密钥（SSCKey）
    处理流程伪代码：

    ``` js
    saltValue = random(); // 使用随机数生成器生成盐值

    // 利用提供的密码熵值、安全存储组件的唯一加密密钥和盐值中导出密码验证器值和密码箱熵值
    (passcodeVerifierValue, lockboxEntropyValue) = func(passcodeEntropyValue, SSCKey, saltValue)

    // 初始化计数器密码箱
    SSC.count = 0；
    SSC.maxRetryTimes = maxRetryTimes; 
    SSC.passcodeVerifierValue = passcodeVerifierValue;  // 保存密码验证器值
    SSC.saltValue = saltValue;                          // 保存盐值

    return lockboxEntropyValue； // 向安全隔区返回密码箱熵值
    ```

### 2. 验证阶段

- 安全隔区：
    发送数据：密码熵值（passcodeEntropyValue）
    接受数据：密码箱熵值（lockboxEntropyValue）
- 安全存储组件：
    自有数据：安全存储组件的唯一加密密钥（SSCKey），密码箱(SSC)
    处理流程伪代码：

    ``` js
    SSC.count++; // 首先增加密码箱的计数器

    if SSC.count > SSC.maxRetryTimes {  // 检查递增的计数器是否超过最大尝试值
        reset(SSC);
        exit -1 ; // 疑似重复攻击，完全清除计数器密码箱并异常退出
    } else {
        // 尝试使用用于创建计数器密码箱的相同算法导出密码验证器值和密码箱熵值
        (passcodeVerifierValue, lockboxEntropyValue) = func(passcodeEntropyValue, SSCKey, SSC.saltValue);
        if passcodeVerifierValu == SSC.passcodeVerifierValue {
            SSC.count = 0;               
            return lockboxEntropyValue; // 向安全隔区返回密码箱熵值并重置计数器
        } else {
            return false;               // 校验不成功，返回错误
        }
    }
    ```

值得注意的是，通过向安全隔区返回密码箱熵值（而非简单的是否），有利于实现与安全存储组件的双向认证，这也是高级别安全的重要特性！

---

## 参考文献

- [Apple平台安全白皮书](https://help.apple.com/pdf/security/en_US/apple-platform-security-guide.pdf)
- [iOS Security Architecture](https://itzone.com.vn/en/article/ios-security-architecture/)
- [蘋果Secure Enclave安全晶片爆硬體漏洞，舊款設備無法修復](https://mrmad.com.tw/secure-enclave-security-chip-explodes-hardware-vulnerabilities)
- [iOS資料保護機制簡介](https://www.kaotenforensic.com/ios/ios-data-protection/#collapse-1-1786)
- [A13 芯片很牛，但是这款神秘的 U1 芯片才是苹果的野心](https://www.infoq.cn/article/zy3kmbnn6d4sgateg4qf)
- [加盐密码哈希：如何正确使用](https://www.tomczhen.com/2016/10/10/hashing-security/)
- [关于Apple Pay十五个技术问题](https://www.beanpodtech.com/%E5%85%B3%E4%BA%8Eapple-pay%E5%8D%81%E4%BA%94%E4%B8%AA%E6%8A%80%E6%9C%AF%E9%97%AE%E9%A2%98%EF%BC%88%E4%B8%80%EF%BC%89/)
- [苹果“芯”基建简史](https://finance.sina.com.cn/stock/hkstock/hkstocknews/2020-06-20/doc-iircuyvi9466114.shtml)
- [MCU 芯片加密历程](https://picture.iczhiku.com/weixin/message1610684573974.html)
- [iOS 安全设计解析（二）：数据加密和保护](https://blog.blupig.net/ios-security-encryption)
- [iOS资料保护机制简介](https://www.kaotenforensic.com/ios/ios-data-protection/)
- [iOS安全体系结构的简要分析](https://github.com/YuZhang/Security-Courseware/blob/master/system-security/ios-security.md)
- [伪随机数生成器（ctr_drbg）的配置与使用](https://blog.51cto.com/u_13640625/3027985)
- [写给开发人员的实用密码学（四）—— 安全随机数生成器 CSPRNG](https://thiscute.world/posts/practical-cryptography-basics-4-secure-random-generators/)
- [CTR-DRBG的缓存侧信道攻击方法](https://securitygossip.com/blog/2020/03/16/pseudorandom-black-swans-cache-attacks-on-ctr-drbg/)
- [了解硬件信任根 - 新思公司](https://www.synopsys.com/zh-cn/china/resources/dwtb/dwtb-cn-q1-21018-rootsoftrusts.html)
- [基于 ARM TrustZone 的 Secure Boot 实现](https://bbs.pediy.com/thread-260399.htm)

### 资料下载

- [apple平台安全白皮书-2021中文版](apple平台安全白皮书-2021中文版.pdf)
- [iOS安全白皮书-2018英文版](iOS安全白皮书-2018英文版.pdf)
