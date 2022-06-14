---
title: Apple数据安全架构分析
date: 2022-06-14 18:21:32
tags:
---

![技术架构](arch.jpg)
![Key](key.png)
![SOC](soc.png)

安全隔区是大多数版本的 iPhone、 iPad、 Mac、 Apple TV、 Apple Watch 和 HomePod 的硬件功能，支持的版本包括 ：

- iPhone 5s 或后续机型
- iPad Air 或后续机型
- 配备触控栏且搭载 Apple T1 芯片的 MacBook Pro 电脑 （2016 年款和 2017 年款）
- 基于 Intel 芯片且搭载 Apple T2 安全芯片的 Mac 电脑
- 搭载 Apple 芯片的 Mac 电脑
- Apple TV HD 或后续机型
- Apple Watch Series 1 或后续机型
- HomePod 和 HomePod mini

安全隔区的基本组件包括：

- 处理器：专门为安全隔区提供计算能力，运行定制版的 L4 微内核
- TRNG：= Hardware Random Number Generator，用于生成安全的随机数据，基于多个环形振荡器并经过 CTR_DRBG 算法处理。
- PKA：用于执行非对称加密操作的专用硬件块，支持 RSA 和 ECC 算法，支持软件密钥和硬件密钥。
- AES引擎：用于实现AES对称加密算法的专用硬件块，支持软件密钥和硬件密钥。硬件密钥派生自安全隔区 UID 或 GID。 这些密钥保存在 AES 引擎内，即使对 sepOS 软件也不可见。 虽然软件可以请求通过硬件密钥进行加密和解密操作， 但不能提取密钥。
- 内存保护引擎：安全隔区从设备 DRAM 内存的专用区域运行，多层保护将由安全隔区保护的内存与应用程序处理器隔离。
- I2C总线：
- EPROM：

在 Apple A14、 M1 及后续型号的 SoC 上， 内存保护引擎支持两组临时内存保护密钥。 第一组密钥用于安全
隔区独有的数据， 第二组密钥用于与安全神经网络引擎共享的数据。
内存保护引擎以内联方式运行且对安全隔区透明。 安全隔区将内存当作普通的未加密 DRAM 一样进行读取和
写入， 而安全隔区外的观察程序只能看到加密和认证版本的内存。 这样做的结果是既提供了强大的内存保护，
又未牺牲性能或增加软件复杂度。

### 根加密密钥

安全隔区包括一个唯一 ID (UID) 根加密密钥。 UID 对于每台设备来说都是唯一的， 且与设备上的任何其他标识符都无关。
随机生成的 UID 在制造过程中便被固化到 SoC 中。 从 A9 SoC 开始， UID 在制造过程中由安全隔区TRNG 生成， 并使用完全在安全隔区中运行的软件进程写入到熔丝中。 这一过程可以防止 UID 在制造过程中于设备之外可见， 因此不可被 Apple 或其任何供应商访问或储存。
sepOS 使用 UID 来保护设备特定的密钥。 有了 UID， 就可以通过加密方式将数据与特定设备捆绑起来。 例如， 用于保护文件系统的密钥层级就包括 UID， 因此如果将内置 SSD 储存设备从一台设备整个移至另一台设备， 文件将不可访问。 其他受保护的设备特定密钥包括触控 ID 或面容 ID 数据。 在 Mac 上， 只有链接到AES 引擎的完全内置储存设备才会获得这个级别的加密。 例如， 通过 USB 连接的外置储存设备或者添加到2019 年款 Mac Pro 的基于 PCIe 的储存设备都无法获得这种方式的加密。

安全隔区还有设备组 ID (GID)， 它是使用特定 SoC 的所有设备共用的 ID （例如， 所有使用 Apple A14 SoC 的设备共用同一个 GID）。
UID 和 GID 不可以通过联合测试行动小组 (JTAG) 或其他调试接口使用。

## 微内核

Mach、L4、seL4微内核OS的简单比较

|微内核OS|通信机制|性能|安全性|可靠性|可拓展性|
|:-:|:-:|:-:|:-:|:-:|:-:|
|Mach|共享内存，同步|☆☆|×|☆☆|☆|
|L4|共享内存，异步|☆☆☆|×|☆☆|☆☆|
|seL4|同步和异步端点|☆☆☆☆|√|☆☆☆|☆☆☆|

苹果突然升级安全隔区，可能与去年出现的 GrayKey 密码破解设备有关系。GrayKey 一次可以连接两台 iPhone，插入 iPhone 后，需要等待 2 分钟安装专门的软件，并可以实现 iPhone 解锁。当软件安装完成后，GrayKey 开始破解密码，这个过程可能耗费几小时，如果密码是 6 位的，也有可能花费几天的时间。GrayKey 属于暴力破解，通过漏洞可以无限次尝试密码，直到密码准确。

正如《iOS 安全指南》所述，Secure Enclave 实际上是苹果对其 A 系列处理器中某个高度机密的称呼，按照 TEE 标准，现在的处理器都包含“普通世界”和“安全世界”两部分，Secure Enclave 就是其中的安全世界。这部分的工作是处理数据保护密钥管理的加密操作 ; 是在通用处理器中区分割出的一个专门处理 Touch ID 指纹、密钥等敏感信息操作的区域。ARM 处理器架构中的 TrustZone 与此相似，或者说苹果的 Secure Enclave 可能就是一个高度定制版的 TrustZone。

苹果在指南中表示，由于 Secure Enclave 与 iOS 其他部分分离，即使内核受到威胁也能保持其完整性。因此系统利用 Secure Enclave 处理 Touch ID 指纹数据，在通过传感器授权的购买行为上签名，通过验证用户的指纹来解锁手机，能够有效地保障安全性。

---

## 参考文献

- [apple平台安全白皮书-2021中文版](apple平台安全白皮书-2021中文版.pdf)
- [iOS安全白皮书-2018英文版](iOS安全白皮书-2018英文版.pdf)
- [SEL4技术白皮书-英文版](seL4-whitepaper.pdf)
- [微内核发展史与Mach、L4、seL4微内核OS的比较](https://blog.csdn.net/xiasli123/article/details/105191368)
- [蘋果Secure Enclave安全晶片爆硬體漏洞，舊款設備無法修復](https://mrmad.com.tw/secure-enclave-security-chip-explodes-hardware-vulnerabilities)
- [iOS資料保護機制簡介](https://www.kaotenforensic.com/ios/ios-data-protection/#collapse-1-1786)
