---
title: ARM TrustZone的技术概述
date: 2022-09-25 18:23:58
tags:
---

## 安全元件 - SE（Secure Element）

19世纪70年代就出现了安全元件Secure Element，其外在表现就是一块物理上独立的芯片卡，负责提供私密信息的安全存储、重要程序的安全执行等功能。

按照Global Platform的定义，SE具有自己独立的执行环境和安全存储，采用安全协议与外部通讯，支持软件防篡改和硬件防篡改，其内部组件包含有：CPU、RAM、ROM、加密引擎、传感器等。
![SE.png](SE.png)

从产品形态上，SE可以分为三种：

- UICC：也称通用集成电路卡，就是电信运营商发布和使用的手机SIM卡
- Embedded SE：嵌入式SE，以一个独立芯片的形式集成在设备的主板上，例如iPhone手机上集成的NXP SN200系列芯片（带NFC功能）
- Micro SD：以SD存储卡的形式存在，通过插入SD卡槽集成到手机上，由独立的SE制造商制造和销售，由于安全和性能问题现已基本淘汰

此外，银行系统的U盾也被认为是SE的一种形态，其核心就是一块智能卡芯片，但是集成了电子数字证书与签名秘钥等更多安全能力。
SE提供了非常可靠的安全等级，但保存在其中的数据和程序需要有更新机制，一般是通过TSM（Trusted Service Manager）来实现的。

## TEE - 可信执行环境

随着智能终端的普及发展，越来越多的终端设备运行在复杂的开放环境之中，仅仅依赖Android等软件技术显然无法提供足够的安全强度，但基于Secure Element的安全技术在集成难度和成本上也存在困难，例如小额支付、版权保护、企业VPN等典型应用的安全保护强度并不高，为此迫切需要在安全性能和成本之间找到平衡点，开发一种“合适强度”的安全架构。
![vs.png](vs.png)

2006年，OMTP工作组智能终端的安全率先提出了一种双系统解决方案：即在同一个智能终端下，除了多媒体操作系统外再提供一个隔离的安全操作系统，这一运行在隔离的硬件之上的隔离安全操作系统用来专门处理敏感信息以保证信息的安全。该方案即TEE的前身。

2013年，Global Platform（GP）提出了可信执行环境（TEE，Trusted Execution Environment）的概念，即不需要独立的物理芯片，而是在通用CPU内核上扩展出的安全可执行模式。

与REE (Rich Execution Environment)对应，TEE是与设备上的Rich OS（通常是Android等）并存的运行环境，通常具有其自身的执行空间，为Rich OS运行某些关键操作（指纹验证、PIN码输入、私钥存储、证书存储、数字版权保护等）提供安全服务，例如：

![TEE.png](TEE.png)

GP在TEE的标准化方面下足了工夫，定义了一系列技术规范：

- Internal API：负责为可信应用TA提供对安全资源和服务的访问，包括密钥注入和管理、加密、安全存储、安全时钟、可信用户界面（UI）和可信键盘等
- Client API：是让运行在Rich OS中的客户端应用（CA）访问TA服务和数据的底层通信接口
- Functional API：包括应用管理、调试功能、安全保护轮廓等

TEE环境比Rich OS(普通操作系统)的安全级别更高，略差于安全元件SE，但成本显著降低，其核心特征是：

- TEE具有其自身的执行空间，也就是说在TEE的环境下有一个独立的操作系统，如seL4微内核
- TEE所能访问的硬件资源是与Rich OS分离的，拥有独立的MCU、DRAM、SRAM等
- TEE提供了授权安全软件(TrustApp可信应用，简称TA)的安全执行环境，同时也保护TA的资源和数据的保密性、完整性和访问权限。为了保证TEE本身的可信根，TEE在安全启动过程中是要通过验证并且与Rich OS隔离的
- 在TEE中，每个TA是相互独立的，而且不能在未授权的情况下互相访问。简而言之就是在TEE环境的操作系统上同样有相应的应用程序(TA)，除了TEE的运行环境与普通操作系统相互独立外，TEE里的每一个TA也是需要授权并相互独立运行的。

## ARM TrustZone

TEE是一种基于通用CPU内核的嵌入式硬件技术，因此不同CPU指令集上必然有着相应的技术实现，包括：

- AMD PSP（Platform Security Processor）处理器
- ARM TrustZone技术（支持TrustZone的所有ARM处理器）
- Intel x86-64指令集：SGX Software Guard Extensions
- MIPS：虚拟化技术Virtualization

ARM架构的处理器是智能终端市场的绝对霸主，也是TEE技术的主导者之一。2006年，ARM提出了硬件虚拟化技术**TrustZone**，通过引入安全扩展成为所有Cortex-A 类处理器的基本功能，随后ARM将其 TrustZone API 提供给 GlobalPlatform，也就是 TEE 的 Client API。

![el3-aarch64.png](el3-aarch64.png)
TrustZone-A引入了安全世界（secure world）和非安全世界（none secure world）的概念，并且能够通过读取SCR（Secure Configuration Register）寄存器的第33位NS（None Secure）比特位进行判断。并且该值可用于外设总线（即通过NS比特位判断哪些设备是安全设备，只能在安全世界进行访问，在硬件层面对外设寄存器的访问进行了限制），以及管控对内存区域的访问（哪片地址空间只能够在安全世界访问，而非安全世界无权访问）。

此外，Cortex-A为CPU引入了新的Monitor Mode模式。Monitor模式是作为安全世界和非安全世界间沟通的存在，无论是从安全世界退出到非安全世界，还是从非安全世界进入安全世界都要经过Monitor模式。从官方的介绍可知，从非安全世界和安全世界进入Monitor Mode有两种方式，一种是通过SMC（Secure Monitor Call）指令，另一种方式是通过在安全世界注册的中断。

![soc.png](soc.png)

需要指出的是，由于Apple、华为、高通等公司的自研CPU芯片都是基于ARM授权，由此Apple的Secure Enclave、高通的QSE等技术就是TrustZone的变种。
例如，华为鲲鹏服务器推出的TrustZone套件，就是一个以ARM TrustZone为技术实现基础，由硬件(包括BIOS、BMC)、机密计算运行环境，以及配套的 patch、应用开发指导和应用打包工具等组成的端到端的解决方案。
![kunpeng.png](kunpeng.png)

---

## 附录一：Android设备的指纹识别

Android设备的指纹识别，依赖TEE来实现用户指纹认证，要求指纹采集、注册和识别都必须在TEE内部进行，已保证安全。

Android从4.0开始引入了KeyStore，开发者可以使用KeyStore API生成密钥、使用密钥签名、使用密钥加解密、获取密钥的属性信息，但无法将密钥本身从KeyStore中取出。因为密钥不进入应用进程，这大大提高了密钥的安全性。随着Android版本更迭，KeyStore的实现不断进化得更加安全，在有些设备上，不仅密钥不进入应用进程，甚至不进入Android OS只存储在TEE或SE中，接下来我们大概列举下KeyStore的进化。

- 4.0：创世版本，密钥使用用户的passcde加密后存储，支持RSA、ECDSA
- 4.1：增加了使用安全硬件的基础设施，在可能的情况下密钥会被存储到安全硬件中
- 6.0：增加支持AES、HMAC；增加了密钥绑定用户认证的能力，即可以指定某些密钥，在每一次使用时，必须由用户进行认证（指纹、passcode等）
- 7.0：强制要求预装7.0系统的设备必须拥有安全硬件并且支持基于安全硬件的KeyStore
- 8.0：增加了设备证明（Key Attestation）能力，开发者可通过验证Key Attestation的证书链，来确认密钥的确保存在了安全硬件中

![finger.png](finger.png)

## 附录二：微内核

与微内核对应的是宏内核，典型产品就是Linux。

宏内核被视作为运行在单一地址空间的单一的进程，内核提供的所有服务，都以特权模式，在这个大型的内核地址空间中运作，这个地址空间被称为内核态（kernel space）。
宏内核通常以单一静态二进制文件的方式被存储在磁盘，或是缓冲存储器上，在引导之后被加载存储器中的内核态，开始运作。
宏内核的优点是设计简单。在内核之中的通信成本很小，内核可以直接调用内核态内的函数，跟用户态的应用程序调用函数一样，因此它的性能很好。在1980年代之前，所有的操作系统都采用这个方式实现；即使到了现在，主要的操作系统也多采用这个方式。

与之相对，微内核的支持者认为，宏内核的移植性不佳，操作系统的代码高度耦合，很难适配不同的CPU架构，此外，所有的模块也都在同一块寻址空间内执行，倘若某个模块有错误，执行时就会损及整个操作系统运作。
微内核的设计理念，是将系统服务的实现，与系统的基本操作规则区分开来。它实现的方式，是将核心功能模块化，划分成几个独立的进程，各自运行，这些进程被称为服务（service）。所有的服务进程，都运行在不同的地址空间。只有需要绝对特权的进程，才能在具特权的执行模式下运行，其余的进程则在用户空间运行。

第一代微内核，在内核提供了较多的服务，因此被称为“胖微内核”，它的典型代表是Mach，它既是GNU HURD也是Mac OS X的内核。
第二代微内核只提供最基本的OS服务，典型的OS是QNX，QNX在黑莓手机BlackBerry 10系统中被采用。L4微内核系列也是著名的微核心

|微内核OS|通信机制|性能|安全性|可靠性|可拓展性|
|:-:|:-:|:-:|:-:|:-:|:-:|
|Mach|共享内存，同步|☆☆|×|☆☆|☆|
|L4|共享内存，异步|☆☆☆|×|☆☆|☆☆|
|seL4|同步和异步端点|☆☆☆☆|√|☆☆☆|☆☆☆|

---

## 参考文献

- [从可信计算到机密计算 - 冯登国](https://zhuanlan.zhihu.com/p/471110552)
- [网银U盾安全认证原理解析](https://zhuanlan.zhihu.com/p/34813040)
- [隐私计算技术之可信执行环境（TEE）](https://zhuanlan.zhihu.com/p/384966998)
- [Arm Trustzone的安全扩展介绍](https://aijishu.com/a/1060000000320633)
- [TrustZone-M简介](https://hack-big.tech/2021/01/23/Microcontroller-TrustZone%E9%9A%94%E7%A6%BB%E6%8A%80%E6%9C%AF%E7%AE%80%E4%BB%8B/)
- [从TrustZone建置安全验证硬件基础 - FIDO联盟](https://www.laoyaoba.com/html/share/news?source=pc&news_id=583957)
- [ARM cortex三个版本A，R, M之间区别](https://www.cnblogs.com/hjbf/p/13298964.html)

### 文档下载

- [创新发展中的可信计算理论与技术 - 冯登国](http://scis.scichina.com/cn/2020/SSI-2020-0224.pdf)
- [安全可信智能移动终端研究 - 张大伟](https://res-www.zte.com.cn/mediares/magazine/publication/com_cn/article/201505/445481/P020151028370420765032.pdf)
- [TEE：从手机端到云端的系统安全增强](https://www.trustkernel.com/uploads/pubs/TEE-%E4%BB%8E%E6%89%8B%E6%9C%BA%E7%AB%AF%E5%88%B0%E4%BA%91%E7%AB%AF%E7%9A%84%E7%B3%BB%E7%BB%9F%E5%AE%89%E5%85%A8%E5%A2%9E%E5%BC%BA.pdf)
- [鲲鹏 BoostKit 机密计算 TrustZone 套件 技术白皮书](https://support.huaweicloud.com/twp-kunpengcctrustzone/twp-kunpengcctrustzone.pdf)
- [SEL4技术白皮书-英文版](seL4-whitepaper.pdf)
