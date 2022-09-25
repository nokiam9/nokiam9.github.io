---
title: Apple数据安全架构分析
date: 2022-06-14 18:21:32
tags:
---

为了确保软件的安全性，它必须安装在具有内建安全性的硬件上，而内建安全性的硬件遵循“**支持有限且单独定义的功能**”的准则，以使攻击面最小化。

![技术架构](framework.png)

- Secure Encalve：Apple定义的安全隔区。按照 TEE 标准，处理器应包含“普通世界”和“安全世界”两部分，Secure Enclave 就是其中的安全世界。ARM 处理器架构中的 TrustZone 与此相似，或者说苹果的 Secure Enclave 就是一个高度定制版的 TrustZone。
- Secure Element：基于Java Card的安全元件，简称SE。iPhone中通常采用第三方的独立芯片，如NXP，一般用于支持基于NFC的Applet应用。
- Device Key：也被称为硬件UID。Apple定义为一个 256 位的 AES 密钥， 在设备制造过程中刻录在每个处理器上。 这种密钥无法由固件或软件读取， 只能由处理器的硬件 AES 引擎使用。 若要获取实际密钥， 攻击者必须对处理器的芯片发起极为复杂且代价高昂的物理攻击。 UID 与设备上的任何其他标识符均无关， 包括但不限于 UDID。

## 安全隔区 - Secure Enclave

SecureEnclave 是 Apple A7 以及后续系列的应用处理器封装在一起的协处理器，它有自己的安全引导过程和个性化的软件更新过程，并且同 iOS 系统所在的应用处理器分离。Secure Enclave 使用加密（使用临时产生的密钥加密）的物理内存（和应用处理器共享的物理内存的一部分空间）进行业务处理，并且有自己的硬件随机数产生器。Secure Enclave 同应用处理器之间通过中断驱动的 mailbox 以及共享内存来进行通信，对外提供数据保护密钥管理相关的所有密码学服务。

ARM TrustZone 技术本质上是一种虚拟化技术，通过将处理器状态分为安全和非安全状态，并且配合其他总线以及外设上的安全属性来实现遍布整个硬件系统的安全。同 Secure Enclave 一样，ARM TrustZone 也有自己的安全引导过程以及个性化的软件更新过程，也有自己的硬件随机数产生器（以及其他类似的安全外设），并且同应用处理器之间也是通过中断驱动的 Monitor 模式以及共享内存来进行通信。可以看出，Secure Enclave 所提供的安全特性并没有超出 ARM TrustZone 技术的范围。

不过就 Apple 的官方信息来说，Apple 从未提过 Secure Enclave 就是 ARM TrustZone 安全扩展技术的实现（虽然根据 Apple 官方文档中关于几个安全通信通道的描述来看，Secure Enclave 很可能是 ARM TrustZone 技术的一种实现），因此我们还是无法断定 Secure Enclave 究竟是独立的协处理器还是应用处理器的一种运行状态（两种架构都可以提供 Secure Enclave 必须的安全特性），这个有待于 Apple 公布更多的 Secure Enclave 的实现细节，就目前而言，可以得出的结论是：Secure Enclave 所提供的安全特性，使用 ARM TrustZone 技术同样可以实现。

2013年，苹果发布的iPhone 5s手机率先提供了指纹锁，为了解决生物特征数据的数据安全问题，采用了其设计软硬件结合的安全方案Secure Enclave。
此后该方案得到了广泛应用，成为后续推出所有iPhone的标配，并为基于Intel CPU的Mac电脑提供了T1、T2芯片，iPad Air、Apple TV、Apple Watch和HomePod等产品线也都支持，更不用后续自研的M1 CPU了。

![SOC](soc.png)

安全隔区是一个独立于操作系统的片上系统（System on Chips），为数据保护密钥管理提供所有加密操作并保持数据保护的完整性（即使内核已被破坏），包含以下基本组件：

- `SEP`：= Secure Enclage Processor，为安全隔区提供专用计算能力，运行Apple定制的L4微内核（也称为sepOS），
- `TRNG`：= Hardware Random Number Generator，基于多个环形振荡器并经过CTR_DRBG算法生成安全的真随机数据
- `PKA`：用于执行非对称加密操作的专用硬件块，支持RSA和ECC算法，支持软件密钥和硬件密钥
- `AES Engine`：用于实现AES对称加密算法的专用硬件块，支持软件密钥和硬件密钥
- `Memory Protection Engine`：安全隔区在设备 DRAM 内存的专用区域运行，多层保护将由安全隔区保护的内存与应用程序处理器隔离
- `I2C bus`：= Inter-Integrated Circuit，集成电路总线，用于读取主机板的安全非易失性存储器，这是Philips开发的一种简单、双向二线制同步串行总线

> AES引擎内部保存了基于UID的硬件密匙，即使对sepOS也不可见
> 虽然安全隔区不含储存设备，但它拥有一套将信息安全储存在所连接储存设备上的机制，该储存设备与应用程序处理器和操作系统使用的 NAND 闪存互相独立

随着Apple芯片的持续演进，安全隔区的保护机制也越来越完善。

![stage](stage.jpg)

- 当设备启动时，Secure Enclave Boot ROM 会创建一个临时内存保护密钥，与设备的UID一起构成加密因子，用于加密 Secure Enclave 的设备内存空间部分
- 在Apple A11之后，完整性树用于防止对安全至关重要的 Secure Enclave 内存的重放，通过内存保护密钥和存储在片上 SRAM 中的随机数进行身份验证
- 在Apple A14 和 M1之后，内存保护引擎支持两组临时内存保护密钥，第一组密钥用于安全隔区独有的数据，第二组密钥用于与安全神经网络引擎共享的数据

安全隔区是一个 (SoC)， 内置于所有新款的 iPhone、 iPad、 Apple Watch、 Apple TV 和 HomePod 设备以及搭载 Apple 芯片和搭载 Apple T2 安全芯片的 Mac 上。 
安全隔区本身 遵循与 SoC 相同的设计准则， 自身拥有独立的 Boot ROM 和 AES 引擎。 安全隔区还为加密静态数据所需 密钥的安全生成和储存提供了基础， 同时还保护和评估触控 ID 和面容 ID 的生物识别数据。
储存加密必须快速高效， 但与此同时不能透露其用于建立加密密钥关系的数据 (或密钥材料)。 AES 硬件引擎 通过在读写文件时执行快速的嵌入式加密和解密解决了这个问题。 安全隔区中的特殊通道向 AES 引擎提供所 需的密钥材料， 而不将此信息透露给应用程序处理器 (或 CPU) 或者整个操作系统。 这有助于确保 Apple 的 数据保护和文件保险箱技术在保护用户文件的同时， 不会透露长期有效的加密密钥。
Apple 所设计的安全启动可保护软件的最底层不被篡改， 并且仅允许来自 Apple 的受信任操作系统软件在启 动时加载。 安全启动在不可更改的代码 (称为 Boot ROM， 在 Apple SoC 制造阶段植入， 也称为硬件信任 根) 中开始执行。 在搭载 T2 芯片的 Mac 电脑上， macOS 安全启动的信任植根在 T2 中。 (T2 芯片和安全 隔区也均会使用各自独立的 Boot ROM 执行自己的安全启动进程， 这准确地模拟了 A 系列芯片和 M1 芯片 安全启动的方式。)
安全隔区还会处理 Apple 设备中分别来自触控 ID 和面容 ID 传感器的指纹和面部数据。 这样做既提供了安 全认证， 同时又保护了用户生物识别数据的隐私和安全， 还可让用户享有较长及较复杂密码带来的安全性， 以 及在许多情形下访问或购买时快速认证的便利性。

Secure Enclave 是在片上系统 (SoC) 中制造的协处理器。它使用加密内存并包含一个硬件随机数生成器。 Secure Enclave 。 Secure Enclave 和应用处理器之间的通信被隔离到一个中断驱动的邮箱和共享内存数据缓冲区。
Secure Enclave 包括一个专用的 Secure Enclave 引导 ROM。与应用处理器 Boot ROM 类似，Secure Enclave Boot ROM 是为 Secure Enclave 建立硬件信任根的不可变代码。
Secure Enclave 运行基于 Apple 定制版本的 L4 微内核的 Secure Enclave 操作系统。此 Secure Enclave OS 由 Apple 签名，，并通过个性化软件更新过程进行更新。
。除了在 Apple A7 上，Secure Enclave 内存也通过内存保护密钥进行身份验证。
在 A11 和更新版本以及 S3 和更新版本的 SOC 上，完整性树用于防止对安全至关重要的 Secure Enclave 内存的重放，通过内存保护密钥和存储在片上 SRAM 中的随机数进行身份验证。
Secure Enclave 保存到文件系统的数据使用与 UID 纠缠的密钥和反重放计数器进行加密。防重放计数器存储在专用的非易失性存储器集成电路 (IC) 中。
在配备 A12 和 S4 SoC 的设备上，Secure Enclave 与用于防重放计数器存储的安全存储集成电路 (IC) 配对。安全存储 IC 设计有不可变 ROM 代码、硬件随机数生成器、加密引擎和物理篡改检测。为了读取和更新计数器，Secure Enclave 和存储 IC 采用了确保对计数器的独占访问的安全协议。

内存保护引擎以内联方式运行且对安全隔区透明。 安全隔区将内存当作普通的未加密 DRAM 一样进行读取和写入， 而安全隔区外的观察程序只能看到加密和认证版本的内存。 这样做的结果是既提供了强大的内存保护，又未牺牲性能或增加软件复杂度。

## 安全密钥体系

![Key](key.png)
![Key](key2.png)

安全隔区包括一个唯一 ID (UID) 根加密密钥。 UID 对于每台设备来说都是唯一的， 且与设备上的任何其他标识符都无关。
随机生成的 UID 在制造过程中便被固化到 SoC 中。 从 A9 SoC 开始， UID 在制造过程中由安全隔区TRNG 生成， 并使用完全在安全隔区中运行的软件进程写入到熔丝中。 这一过程可以防止 UID 在制造过程中于设备之外可见， 因此不可被 Apple 或其任何供应商访问或储存。
sepOS 使用 UID 来保护设备特定的密钥。 有了 UID， 就可以通过加密方式将数据与特定设备捆绑起来。 例如， 用于保护文件系统的密钥层级就包括 UID， 因此如果将内置 SSD 储存设备从一台设备整个移至另一台设备， 文件将不可访问。 其他受保护的设备特定密钥包括触控 ID 或面容 ID 数据。 在 Mac 上， 只有链接到AES 引擎的完全内置储存设备才会获得这个级别的加密。 例如， 通过 USB 连接的外置储存设备或者添加到2019 年款 Mac Pro 的基于 PCIe 的储存设备都无法获得这种方式的加密。

安全隔区还有设备组 ID (GID)， 它是使用特定 SoC 的所有设备共用的 ID （例如， 所有使用 Apple A14 SoC 的设备共用同一个 GID）。
UID 和 GID 不可以通过联合测试行动小组 (JTAG) 或其他调试接口使用。

## 数据安全

![aaa](key.png)

内置两个初始密钥 unique ID (UID) and a device group ID (GID) are AES 256-bit keys。这两个东西谁都无法读写，除了加密引擎。

### UID

The UID是每个设备独有的 and  not recorded by Apple or any of its suppliers.（革命形势啊！）
由于UID是每个设备独有的，而且它是整个ios系统密钥树的最顶层，因此，数据就绑定了设备，flash、memory离开了根，就无法解密.
安全隔区包括一个唯一 ID (UID) 根加密密钥。 UID 对于每台设备来说都是唯一的， 且与设备上的任何其他标识符都无关。
随机生成的 UID 在制造过程中便被固化到 SoC 中。 从 A9 SoC 开始， UID 在制造过程中由安全隔区TRNG 生成， 并使用完全在安全隔区中运行的软件进程写入到熔丝中。 这一过程可以防止 UID 在制造过程中于设备之外可见， 因此不可被 Apple 或其任何供应商访问或储存。
sepOS 使用 UID 来保护设备特定的密钥。 有了 UID， 就可以通过加密方式将数据与特定设备捆绑起来。 例如， 用于保护文件系统的密钥层级就包括 UID， 因此如果将内置 SSD 储存设备从一台设备整个移至另一台设备， 文件将不可访问。 其他受保护的设备特定密钥包括触控 ID 或面容 ID 数据。 在 Mac 上， 只有链接到AES 引擎的完全内置储存设备才会获得这个级别的加密。 例如， 通过 USB 连接的外置储存设备或者添加到2019 年款 Mac Pro 的基于 PCIe 的储存设备都无法获得这种方式的加密。

### GID

The GID 是一类芯片一个，比如A5的芯片内置的GID密钥都一样。
安全隔区还有设备组 ID (GID)， 它是使用特定 SoC 的所有设备共用的 ID （例如， 所有使用 Apple A14 SoC 的设备共用同一个 GID）。


UID 和 GID 不可以通过联合测试行动小组 (JTAG) 或其他调试接口使用。

---

## 附录一：Apple公司的芯片系列

### 主处理器芯片

- A系列：智能终端的iOS，使用于iPhone、iPad、iPod touch、Apple TV及Studio Display产品线。从A4开始，最新型号是 iPhone 13 搭载的 A15
![A](apple-A.jpeg)

- S系列：可穿戴设备的Watch OS，使用于Apple Watch和HomePod Mini产品线，最新型号是S7
![S](Apple-S.jpeg)

- M系列：生产力工具的Mac OS，使用于Mac OS的笔记本电脑和台式机，基于ARM架构的自研处理器，包括M1、M2等
![M](apple-M1.jpeg)

## 功能芯片

- W系列：蓝牙芯片，用于AirPods第二代及之后的蓝牙无线耳机，替换早期的W系列。
- U系列：定位芯片，用于UWB（ultra-wide band）精确定位的芯片，替代已被放弃的基于低功耗蓝牙BLE的iBeacons技术，2019年首次出现在iPhone11

### 已停止发展

- T系列：用于基于Intel芯片的Mac电脑的安全芯片，包括T1和T2，该系列已停止并整合进M系列
- W系列：用于蓝牙耳机Airpods，2016首发包含W1芯片，后续还有W2、W3，随后被重新命名为H系列，以及被集成到S系列芯片。





---

## 参考文献

- [Apple平台安全白皮书](https://help.apple.com/pdf/security/en_US/apple-platform-security-guide.pdf)
- [iOS Security Architecture](https://itzone.com.vn/en/article/ios-security-architecture/)
- [蘋果Secure Enclave安全晶片爆硬體漏洞，舊款設備無法修復](https://mrmad.com.tw/secure-enclave-security-chip-explodes-hardware-vulnerabilities)
- [iOS資料保護機制簡介](https://www.kaotenforensic.com/ios/ios-data-protection/#collapse-1-1786)
- [A13 芯片很牛，但是这款神秘的 U1 芯片才是苹果的野心](https://www.infoq.cn/article/zy3kmbnn6d4sgateg4qf)
- [加盐密码哈希：如何正确使用](https://www.tomczhen.com/2016/10/10/hashing-security/)
- [关于Apple Pay十五个技术问题](https://www.beanpodtech.com/%E5%85%B3%E4%BA%8Eapple-pay%E5%8D%81%E4%BA%94%E4%B8%AA%E6%8A%80%E6%9C%AF%E9%97%AE%E9%A2%98%EF%BC%88%E4%B8%80%EF%BC%89/)

### 资料下载

- [apple平台安全白皮书-2021中文版](apple平台安全白皮书-2021中文版.pdf)
- [iOS安全白皮书-2018英文版](iOS安全白皮书-2018英文版.pdf)
