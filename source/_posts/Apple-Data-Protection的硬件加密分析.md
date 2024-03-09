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

## 附录一：FIPS 140-2

FIPS 140 是 NIST（国家标准与技术研究院，美国）制定的一项技术标准，描述了用于敏感但非保密 (SBU) 用途的 IT 产品所需满足的加密和相关安全要求。
1994年发布了 FIPS 140-1，2001年发布了 FIPS 140-2 为**现行标准**，2009年发布了一份 FIPS 140-3 的草案，但目前没有任何产品通过验证。

FIPS 140-2 定义了 4 个安全等级，包括：

- L1：通常用于仅软件加密产品。要求验证至少一个已批准的加密演算法或安全功能，并确保所有组件符合生产级评估要素。
- L2：要求提供**基于角色的验证**，并具有通过使用物理锁定或防篡改签章识别物理篡改的能力。
- L3：要求提供**基于身份的认证**，并提供物理篡改预防措施，防止拆卸或修改，如果检测到篡改，设备必须能够擦除关键安全参数，要求私钥只能以加密形式进入或离开等。
- L4：专用于缺少物理保护的环境，要求提供高级篡改保护，如果检测到各种形式的环境攻击，则擦除设备的内容。

美国和加拿大的法律规定，政府采购项目必须使用 FIPS 140-2 2 级验证产品，NIST 提供所有可用于商业的[FIPS 140-2 认证产品清单](https://csrc.nist.gov/projects/cryptographic-module-validation-program)。

> CAVP （通用）密码算法验证体系 + CMVP （产品）密码模块验证体系 = FIPS 140-2

## 附录二：国际 CC 信息安全认证体系

1993 年，在美国的 TCSEC、欧洲的 ITSEC、加拿大的 CTCPEC、美国的 FC（Federal Criteria）等信息安全准则的基础上，由 6 个国家 7 方(美国国家安全局 NSA 和国家技术标准研究所 NIST、加、英、法、德、荷)共同提出了**信息技术安全评价通用准则**（The Common Criteria for Information Technology security Evaluation，简称 CC 标准），目标是采用一套公共的准则规范为 IT 保障类产品的广泛用户群体提供最大的便利，提供了一个更全面的框架用来评估信息系统、信息产品的安全性。

![历史](CC-his.png)

### TOE & PP & ST

CC 相当于一个标准化的安全需求目录，它并非针对具体的产品，具有较大的灵活性。

- PP（Protection Profile，保护轮廓）：满足特定用户需求，与一类 TOE 实现无关的一组安全要求。
- ST（Security Target，安全目标）：作为指定的 TOE 评估基础的一组安全要求和规范。
- 评估对象（Target of Evaluation，TOE）：作为安全评估对象的具体信息技术产品(如防火墙、计算机网络、密码模块等)，包括相关的管理员指南、用户指南、设计方案等文档。

总的来说，PP 相当于行业的一个规范或标准，它是对标整个行业的，表达“要求产品做成什么样子”；ST 是针对具体厂商 而言的，表达“产品想做成什么样子”，相当于产品的安全功能说明书；TOE 则是某个厂商的具体产品。

### EAL 1 - 7

EAL（Evaluation Assurance Level，评估保护等级）描述了评估深度和严谨程度的数字评级，每级 EAL 都定义了 6 项SAR（Security Assurance Requirements，安全保证要求），包括：ST 的评估准则、TOE 的开发、生命周期支持、指导性文件、测试、脆弱性评定。

- EAL 1：功能测试，**基础级别**的安全保障，适用于对正确运行需要一定信任的场合，安全威胁应视为并不严重。
    依据一个规范的独立性测试和对所提供指导性文档的检查来为用户评估 TOE；
    无需开发者配合即可完成评估，仅确认 TOE 的功能与其文档在形式上是一致的，且对已标识的威胁提供了有效的保护。
- EAL2：结构（白盒）测试，适用于缺乏现成可用的完整的开发记录时，开发者或用户需要一种**低等到中等级别**的、独立保证的安全性。
    增加开发者测试及脆弱性分析和基于 TOE 规范的独立性测试；
    提供功能和接口的规范、指导性文档和TOE的高层设计的评估分析；
    要求开发者递交设计信息和测试结果，但不需要开发者增加过多费用或时间投入；
    评估证据包括安全功能的独立性测试、开发者自测结果确认、功能强度分析、脆弱性报告、TOE 的配置表和安全交付程序；
- EAL3：系统地测试和检查，适用于开发者或用户需要一个**中等级别**的、独立保证的安全性，且在不带大量重建费用的情况下，对 TOE 及其开发过程进行彻底审查。
    增加更完备的安全功能测试，及提供开发过程中不会被篡改的可行性机制或程序；
    增加开发环境现场核查，核查配置管理、 开发过程管理、交付过程管理等。
- EAL4：系统地设计、测试和复查，适用于开发者或用户对传统的商品化的 TOE 需要一个**中等到高等级别**的独立保证的安全性，并且准备负担额外的安全专用工程费用。
    增加更多的设计描述、实现的子集，以及提供开发或交付过程中不会被篡改的可信性改进机制或程序；
    增加对抵御攻击的能力的独立脆弱性分析、开发环境控制措施（如自动化的额外的配置管理和安全交付程序）、源码审查；
    需要分析 TOE 模块的低层设计和实现的子集；
- EAL5：半形式化设计和测试，**高等级别**，适用于开发者和使用者在有计划的开发中，采用严格的开发手段，以获得一个高级别的独立保证的安全性，不会因采取专业性安全工程技术而增加一些不合理的开销。
    需要分析所有的实现，还需要额外分析功能规范和高层设计的形式化模型和半形式化表示和论证；
- EAL6：半形式化验证的设计和测试，适用于在**高风险**环境下的特定安全产品或系统的开发，且要保护的资源值得花费一些额外的人力、物力和财力。
- EAL7：形式化验证的设计和测试，适用于一些安全性要求很高的 TOE 开发，这些 TOE 将应用在风险非常高的地方，或者所保护资产的价值很高的地方，非常罕见！

![EAL](EAL.png)

### 与 ISO 的关系

2005年，国际标准组织（ISO）正式采纳 CC 2.3 版本为 ISO/IEC 15408。
2008年，中国也引入了 ISO/IEC 标准作为 GB/T 18336 国家标准，也是基于CC 2.3版本。
当前 CC 标准的最新版本为 CC v3.1，请参见[https://www.commoncriteriaportal.org](https://www.commoncriteriaportal.org/cc/index.cfm)

目前 CCRA 互认的安全保障级别 EAL 最高为 4 级，为此该协定组织明确规定了通用标准评估方法论（Common Evaluation Methodology，CEM）作为互认协定所使用的标准基础，并被 ISO 组织接纳为国际标准 ISO/IEC 18405。

### CCRA 国际互认协定

1998 年，标准开发组的参与国联合了其他国家共同签署了 CC 互认协定（Common Criteria Recognition Arrangement，CCRA），目前包括美国、英国、德国、日本、韩国和澳大利亚等26个国家。
CCRA 各个成员国家的 CC 评估和认证体系均设有一个权威的认证机构和多个获得商业授权的评估实验室，他们与评估发起者（申请者）合作完成产品的评估和认证，权威认证机构接受所在国家政府职能机构和 CCRA 的双重监管。

在美国，NIAP（National Information Assurance Partnership，美国国家信息保障合作组织）是 NSA（美国国家安全部）的一个部门，负责美国 CC 标准的实施。
CCEVS（Common Criteria Evaluation and Validation Scheme，通用标准评估与认证体系）是 NIAP 管理的一个国家计划，成立于 2009 年，核心任务是评估信息技术的安全功能是否符合 CC 国际标准。
[当前的认证产品列表](https://www.niap-ccevs.org/Product/index.cfm)显示，Cisco、Microsoft、RedHat、Nokia等公司都在其列。

在德国，CC 评测认证体系由德国 BSI（The Bundesamt fur Sicherheit in der Informationstechnik）负责开展，其是德国联邦政府的 IT 安全权威部门。

> 目前，中国没有参加 CCRA 组织。

---

## 参考文献

- [CC 标准 - Wiki](https://en.wikipedia.org/wiki/Common_Criteria)
- [Apple安全密钥存储的加密模块](Apple-Secure-Key-Store-Cryptographic-Module.pdf#12)

### 文档下载

- [ICT 产品信息安全北美市场准入体系研究报告](ICT-report.pdf)

