---
title: FIPS 140 认证体系
date: 2024-03-10 23:42:26
tags:
---

## 一、概述

FIPS 140 是 NIST（国家标准与技术研究院，美国）制定的一项技术标准，描述了用于敏感但非保密 (SBU) 用途的 IT 产品所需满足的加密和相关安全要求。

1994年发布了 FIPS 140-1，2001年发布了[FIPS 140-2](https://csrc.nist.gov/pubs/fips/140-2/upd2/final)成为业界主流标准 ，2019年正式发布了[FIPS 140-3](https://csrc.nist.gov/pubs/fips/140-3/final)，其管理方式有重大变化，不再直接定义模块要求，而是引用了 ISO/IEC 的技术标准，其中加密模块的安全要求遵循 ISO/IEC 19790，加密模块的测试要求遵循 ISO/IEC 24759。

2020年，CMVP 开始接受 FIPS 140-3 的验证申请，但实施过程出现严重积压，截至目前还没有任何产品获得通过，预计 FIPS 140-2 至少延期至 2026 年。

## 二、安全等级

FIPS 140-2 定义了 4 个安全等级，包括：

- L1：通常用于仅软件加密产品。要求验证至少一个已批准的加密演算法或安全功能，并确保所有组件符合生产级评估要素。
- L2：要求提供**基于角色的验证**，并具有通过使用物理锁定或防篡改签章识别物理篡改的能力。
- L3：要求提供**基于身份的认证**，并提供物理篡改预防措施，防止拆卸或修改，如果检测到篡改，设备必须能够擦除关键安全参数，要求私钥只能以加密形式进入或离开等。
- L4：专用于缺少物理保护的环境，要求提供高级篡改保护，如果检测到各种形式的环境攻击，则擦除设备的内容。

## 三、认证体系

[CAVP（Cryptographic Algorithm Validation Program，加密算法验证计划）](https://csrc.nist.gov/projects/cryptographic-algorithm-validation-program)由 NIST 于 1995 年 7 月建立，旨在根据 FIPS/NIST/CSE 推荐的加密算法及算法组件展开验证测评，验证测试包括分组密码、分组密码模式、数字签名、密钥管理、信息验证、随机数列生成、安全散列等。
加密算法验证是加密模块验证程序(CMVP)下的 FIPS140-2 验证的先决条件。

[CMVP（Cryptographic Module Validation Program，加密模块验证计划）](https://csrc.nist.gov/projects/cryptographic-module-validation-program)由美国 NIST 和加拿大政府的通讯安全组织(Communications security establishment, CSE)于 1995 年共同建立，其目标是提供一份可供采购使用的 IT 安全产品列表，列表上的产品已成功通过 FIPS 140-2 标准验证。

所有基于 CMVP 的评测由第三方的授权实验室展开，这些实验室被 NVLAP（National Voluntary Laboratory Accreditation Program，国家实验室自愿认可体系）授权为 CST（Cryptographic and Security Testing，密码和安全测评）实验室，对验证测试感兴趣的供应商可以选择 21 个实验室中的任何一个。

![FIPS 140-2](FIPS140-2.jpg)

美国和加拿大的法律规定，政府采购项目必须使用 FIPS 140-2 2 级验证产品，NIST 提供所有可用于商业的[FIPS 140-2 认证产品清单](https://csrc.nist.gov/projects/cryptographic-module-validation-program)。

## 四、简要分析

FIPS 140 和 CC 标准是两个互补、但不同的安全标准。 FIPS 140 主要提供**加密功能验证**，而 CC 标准的评估范围包含了 IT 产品中更广泛的安全功能选择。 CC 标准评估可能依赖于 FIPS 140 验证来保证基本加密功能已正确实现。

---

## 参考文献

- [从美国FIPS产品体系浅窥我国密码发展趋势](https://www.secrss.com/articles/7687)

### 文档下载

- [NIST FIPS 140-2 加密模块的安全要求，2001年](NIST.FIPS.140-2.pdf)
- [NIST FIPS 140-3 加密模块的安全要求，2019年](NIST.FIPS.140-3.pdf)
