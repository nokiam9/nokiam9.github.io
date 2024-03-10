---
title: Apple Data Protection的硬件加密分析
date: 2024-03-04 21:23:05
tags:
---

根据 Apple 官网提供的[第三方平台认证信息](https://support.apple.com/zh-cn/guide/certifications/welcome/web)，其硬件、操作系统、应用软件、互联网服务和 Apple Pay 均符合 FIPS 140-2 认证，但 FIPS 140-3 认证仍在进行中，[安全隔区的加密模块报告](Apple_Secure_Key_Store_Cryptographic_Module_v10.0.pdf)的最新版本是 v10.0。
此外，Apple 也符合通用标准 (CC) 认证，并委托知名评估机构 [ATSEC 公司](https://www.atsec.com)出具了[iOS 16 安全评估报告 TOE - Target of Evaluation](https://www.niap-ccevs.org/MMO/Product/st_vid11349-st.pdf)，当前版本是 v1.1。
本文就是依据上述公开信息的研究分析。

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

---

## 参考文献
