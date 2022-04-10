---
title: 超级SIM卡的技术标准分析
date: 2021-07-10 15:12:21
tags:
---

随着IC卡从简单的同步卡发展到异步卡，从简单的EPROM卡发展到内带微处理器的智能卡(又称CPU卡)，对IC卡的各种要求越来越高。而卡本身所需要的各种管理工作也越来越复杂，因此就迫切地需要有一种工具来解决这一矛盾，而内部带有微处理器的智能卡的出现，使得这种工具的实现变成了现实。

Chip Operating System（片内操作系统，简称COS），就是基于智能卡内置的微处理器芯片的操作系统。COS的出现不仅大大地改善了智能卡的交互界面，使智能卡的管理变得容易；而且，更为重要的是使智能卡本身向着个人计算机化的方向迈出了一大步，为智能卡的发展开拓了极为广阔的前景。

在这一领域处于领导地位的有三项主导技术：`Java Card`、`Windows Powered Smart Cards`以及`MULTOS`，但从发展情况看，JavaCard已经成为事实上的行业标准。

与Windows、Unix等通用操作系统不同，COS所需要解决的主要还是对外部的命令如何进行处理、响应的问题，这其中一般并不涉及到共享、并发的管理及处理，而且就智能卡在应用情况而看，并发和共享的工作也确实是不需要的；此外，由于 COS 不可避免地受到了智能卡内微处理器芯片的性能及内存容量的影响，系统设计时一般都是紧密结合智能卡内存储器分区的情况，按照国际标准（ISO/IEC7816系列标准）中所规定的一些功能进行裁剪或自定义扩充，因此COS很大程度上是一个专用系统而不是通用操作系统。

## JavaCard

### 概述

多年以前，Sun Microsystem 就非常重视智能卡的市场空间，对 Java技术规范进行了大量裁剪和简化，为智能卡提供了 JavaCard平台，主要目的是在智能卡或与智能卡相近的设备上，以具有安全防护性的方式来运行小型的Java Applet。

JavaCard就是在智能卡ROM中实现了一个Java虚拟机（Java Virtual Machine 简称JVM），该 JVM将执行一个Java字节码的子集，提供外部可以访问的功能，负责控制对智能卡资源的访问(如内存和I/O)。Javacard用于使用Java编程语言以及JVM和Java库的有限版本，用于编写智能卡平台的应用程序-javacard applet。

Java Card虚拟机（Java Card Virtual Machine，也可简称为Java Card VM或JCVM），是原有Java虚拟机的子集合，负责对Java Applet进行程序直译、运行及结果回应，也因此JCVM的空间占量不能太大，必须能小到放入智能卡内。

### 基础架构

{% asset_img javacard.png %}

一台典型的 JavaCard 设备有一个运行于 3.7MHz的8位或16位CPU，带有 1K的RAM和多于 16K的非易失内存（EEPROM或闪存）。高性能的智能卡带有单独的处理器、加密芯片和内存加密，某些智能卡还带有32位CPU。

既然有容量取向的要求，那也就必须对Java的功效机能进行部分权衡取舍，即便可以用多种方式让应用程序的体积占量突破容量限制，例如将应用程序的代码划分到Package（Java编程语言中，用来将类以性质、用途等不同取向等而集中放置的地方，即称为Package）内，但是每个Package也被限制不能超过64KB的容量。

由于Java Card的应用程序是在Java Card VM具隔离性的环境下运行，所以程序对卡片资料的写入、读取、修改也受到权限机制的控制保护，无论使用何种读卡设备、操作系统、应用程序都不能跨越权限去访问不属于自己的卡片内资料，等于具有小型应用程序的防火墙的功效。 无论是电信方面还是金融方面的智能片应用，现在都运用Java Card技术来防护卡内所存储的信息资料。

与以前的智能卡技术相比，采用JavaCard技术的智能卡可以采用 Java进行编程，因此具有高度的可移植性和安全性，更为重要的是，在卡被发送到用户以后，应用程序还可以被动安全地下载到卡中，从而动态地扩展了智能卡的功能。

### 技术组件

JavaCard提供了 JavaCard字节码验证方案，并支持代码签名的安全模型，这意味着在使用由`OpenPlatform`规定的安全下载应用过程下载应用程序之前，必须先将程序代码经过安全的转换 、评估和签名。

Java Card技术规范的最新版本为`3.1`, 但目前应用最广泛的是版本`2.2`，由三部分组成：

- Java Card 虚拟机规范，定义了用于智能卡的 Java 程序编程语言的一个子集和虚拟机。
- Java Card 运行时环境规范，详细定义了基于 Java 的智能卡的运行时行为。
- Java Card API 规范，定义了用于智能卡应用程序的核心框架和扩展 Java 软件包和类。

Sun 还提供了 Java Card 开发工具（JCDK），其中包括 Java Card RE 和 Java Card VM 的参考实现以及其他帮助开发的 Java Card applet 。

## GlobalPlatform

1999年10月，Visa将其建立的Visa Open Platform（简称VOP）更名为GlobalPlatform（GP）并对外开放，使其成为一个由支付与商业领域的大公司、政府部门以及卖主团体主导的、跨行业的国际标准组织，这是世界上第一个跨行业的智能卡规范组织，目标就是促进多应用产业环境的管理及其安全、可互操作的业务部署。这样的话，智能卡发行商将有在各种卡、终端及后台系统中选择的自由。

作为全球基于安全芯片的安全基础设施统一标准的制定者，GlobalPlatform的工作重心主要集中在安全单元（SE）、可信执行环境（TEE）和系统消息（Mobile Messaging）等领域，其成熟的技术规范是建立端到端可信业务解决方案的工具，并服务于产业环境的多个成员，支持多种商业模式。

GP在全球有120多家会员企业，截至2017年，基于GP规范的安全元件出货量就已超过50亿。2015-2017年，搭载SE的移动设备达到10亿+，其全部都是基于GP的标准。

### 卡片的技术架构

{% asset_img gp-card-arch.png %}

GlobalPlatform是一个用于管理applet感知智能卡的规范，用于定义以下内容的操作:

- 管理卡生命周期
- 卡/主机认证
- 安装/删除/实例化/选择小程序
- 管理卡上的安全策略

使用GlobalPlatform, 您将使用GP卡交换APDU以进行上述操作; 使用javacard, 您将编写可以接受和处理特定于您的应用程序的APDU的applet。
GlobalPlatform 并不特定于javacard,但javacard是智能卡applet开发的唯一相关技术.

所有这些应用必须在一个安全的runtime环境中实现，runtime环境提供了一套硬件中立的应用编程接口以支持应用的可移植性。 GlobalPlatform并不强制规定运行时环境的实现技术。卡片管理器作为GlobalPlatform架构中的首要组件起到了 GlobalPlatform卡片中心管理者的作用，特定的密钥和安全管理应用被称作安全域，负责确保发卡方和其他安全域提供者之间的密钥的完全隔离。

GP当前最新版本是v2.3。

### 安全的技术架构

{% asset_img gp-sec-arch.png %}

作为卡外授权机构的卡片内代表的安全域，依据现有的三种授权机构，可以划分为三种主流类型：

- 发卡方安全域（主安全域，ISD），卡片上首要的、强制性存在的安全域，是卡片管理者(通常是发卡方)在卡片内的代表；
- 补充安全域（辅助安全域，SSD），卡片上次要的、可选择地存在的安全域，是应用提供方或发卡方以及它们的代理方在卡片内的代表；
- 授权管理者安全域，一种特殊类型的补充安全域，授权管理者负责将某种安全策略贯彻到所有加载到卡片的应用代码上，授权管理者安全域就是授权管理者在卡片内的代表，卡片上可能存在多个这样的安全域。

总而言之，以上三种安全域在本规范中，统称安全域。
安全域负责提供各类安全服务，包括密钥管理、加密解密、针对其提供者(发卡方、应用提供方、授权管理者)的应用进行数字签名的生成与验证。
当发卡方、应用提供方、授权管理者等卡外实体要将用到的密钥从其他实体区隔开来时，就可以通过新的安全域来代理它们实现这个需求。

## JCOP

最初，Java Card OpenPlatform (JCOP)是一个来自 IBM 苏黎世实验室的产品，目标是整合Javacard API和GlobalPlatform，以提供一个智能卡操作系统及完整的解决方案。
从2007年，JCOP成为恩智浦公司（NXP，曾经是Philips的子公司）拥有和管理，成为NXP JCOP系列芯片卡的标准操作系统。

> 事实上，国内大多数SIM卡厂家都是直接采用NXP芯片和JCOP操作系统，甚至iPhone手机的SE芯片也是NXP提供的。

NXP JCOP系列芯片卡是恩智浦NXP公司在高安全性的解决方案高性能产品，广泛应用如银行与金融，移动通信，公共交通，访客访问和网络接入等领域。支持接触式、非接触式、支持接触式与非接触式读写，内含有一个JCOP版本操作系统，并提供40k-80K字节EEPROM存储器。

当前最新版本是v4，其主要技术特性包括：

- Java Card v3.0.5 Classic
- GlobalPlatform®
  - GP v2.2 Mapping Guidelines configuration v1.0.1
  - GP v2.3 Financial Configuration v1.0 (config 2)
  - GP v2.3 Common Implementation Configuration v2.0 - Common Criteria EAL 5+ certified (supports 3DES,
  RSA, AES)
- ISO 7816-3 T=0, T=1 (223.2 kbps)，用于接触式智能卡
- ISO 14443 (up to 848 kbps)，用于非接触式智能卡
- Dual-interface support，同时支持接触式智能卡/非接触式智能卡的接口

---

## 参考文献

### 官方网站

- [Sun JavaCard 3.1的官方网站](https://docs.oracle.com/en/java/javacard/3.1/)
- [JavaCard SDK的官方版本下载页](https://www.oracle.com/java/technologies/javacard-downloads.html)
- [GlobalPlatform Card的官方文档](https://globalplatform.org/specs-library/card-specification-v2-3-1/)
- [Wiki of JavaCard](https://en.wikipedia.org/wiki/Java_Card)
- [Wiki of JCOP](https://en.wikipedia.org/wiki/Java_Card_OpenPlatform)
- [用于电子支付的基于近距离无线通信的移动终端安全技术要求(GB/T 34095-2017 )](http://openstd.samr.gov.cn/bzgk/gb/newGbInfo?hcno=23B5E4704F5F2752BB0F4BD5FB33E2BC)

### 经验分享

- [NFC中文资料大全！！！](https://wiki.nfc.im/)
- [智能卡操作系统COS概述](https://blog.csdn.net/songbohr/article/details/6201956)
- [JavaCard开发经验合集](https://blog.csdn.net/qq_29605685/article/details/53127366)
- [PBOC规范下的java卡介绍](https://www.msrfid.com/Service/javacard_introduction_under_PBOC_specification.html)
- [GP规范中文版的简介](https://blog.csdn.net/zlljsf1/article/details/4301294)
- [GP规范中定义的四种SE访问控制架构](https://cloud.tencent.com/developer/article/1170054)
- [NXP JCOP系列芯片卡特点](https://www.geek-share.com/detail/2709904440.html)

### 文档下载

- [智能卡技术的新发展 - JavaCard技术综述 - 西安交大 李增智](JavaCard技术综述-西安交大李增智.pdf)
- [Chip Operating System的技术简介](3736-05-open-platform-smart-card-cn.pdf)
- [GlobalPlatform Card SpecificationVersion v2.3.1 - PDF](GPC_CardSpecification_v2.3.1_PublicRelease_CC.pdf)
- [恩智浦 JCOP v4 技术白皮书](NXP-JCOP-4.pdf)
- [用于电子支付的基于近距离无线通信的移动终端安全技术要求（GB征求意见稿）](20140415141748279422.pdf)
- [龙杰智能卡的产品说明书](FSP-ACOSJ-G-Combi-CN-2.04.pdf)
