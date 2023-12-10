---
title: GCM（Galois/Counter Mode）认证加密模式
date: 2023-12-02 17:29:03
tags:
---

## 一、背景

研究分组密码工作模式的核心是**机密性保护（Confidentiality）**，历史上曾经将**完整性保护（Integrity**，例如在某些数据被修改后的情况下密码的误差传播特性）也一并纳入设计目标，但在发现兼顾两个设计目标的难度之后，密码学社区已经普遍将完整性保护作为另一个完全不同的，与加密无关的密码学目标，并开始研究专门的认证加密模式（AE，Authenticated Encryption）。

在密码学中，消息验证码（MAC，Message Authentication Code，也称为标签Tag）是用于验证消息的一小段信息。换句话说，用于确认消息来自指定的发送者（其真实性）并且没有被篡改了。
一般情况下，消息认证码由三种算法组成：

- 密钥生成算法：从密钥空间中随机均匀地选择一个密钥。
- 签名算法：有效地返回给定密钥和消息的标签。
- 验证算法：有效地验证给定密钥和标签的消息的真实性。 也就是说，当消息和标签没有被篡改或伪造时，返回被接受，否则返回被拒绝。

针对消息验证码，NIST 先后提出了 HMAC，CMAC 和 GMAC。
2002年，HMAC 通过认证为[NIST FIPS 198-1](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.198-1.pdf)，基于单向散列函数实现，可以检查消息的完整性，但无法确保来源的可靠性。
2005年，CMAC 标准化为[NIST SP800-38B](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-38b.pdf)，基于分组密码实现，基于共享密钥派生出两个中间密钥，并进行两次哈希计算得到。
2007年，GMAC 标准化为[NIST SP800-38D](nistspecialpublication800-38d.pdf)，基于伽罗华域乘法实现，可以看做对称加密算法和消息认证码的结合。

## 二、基本流程

GCM（Galois/Counter Mode）是一种基于对称加密算法的认证加密（Authenticated Encryption）模式，用于确保数据的机密性和完整性。GCM 结合了 CTR（Counter）模式用于加密和 GHASH（Galois Hash）用于认证，因此不仅继承了 CTR 模式优秀的并行计算能力，还能额外提供对消息完整性、真实性的验证能力，因此广泛应用于 TLS、IPsec 等安全协议中。

根据[NIST SP800-38D](nistspecialpublication800-38d.pdf)的定义，其认证加密过程分为三个阶段：

1. **初始化阶段**：
    根据给定的密钥（key）和初始向量（IV）生成一个初始计数器 ICB；
    计算计数器的初始值J0，当 IV 长度为96位时，J0 = IV || 0^31 || 1。
2. **加密阶段**：
    使用 GCTR（Galois Counter）模式对明文进行加密，通过将计数器块（CB）与密钥（key）进行加密，然后将结果与明文异或，生成密文。
    计数器每次递增，直至加密整个明文。这里，CB 的初始值是 J0 的递增值（即 J0 + 1）。
3. **认证阶段**：
    首先，使用密钥（key）计算一个值H（称为GHASH_H），用于 GHASH 运算。
    接着，将关联数据AD 和密文一起输入到 GHASH 算法中，生成一个认证标签。

![GCM](GCM.png)

解密过程与加密过程类似，首先使用 GHASH 验证密文的认证标签。如果认证通过，再使用 GCTR 模式解密密文，还原成明文。

## 三、核心算法

### 1. 有限域乘法运算 $Z=X \otimes Y$

基于有限域$GF(2^{128})$，定义本原多项式=$x^{128}+x^7+x^2+1$。
处理逻辑：略。

### 2. 带密钥的哈希值计算 $Y=GHASH_H(X)$

先决条件：哈希算法的子密钥$H$
输入参数：128位比特流$X$
输出结果：分组$Y$
核心逻辑：

$Y=(X_1 \otimes H^m) \oplus (X_2 \otimes H^{m-1}) \oplus ... \oplus (X_{m-1} \otimes H^2) \oplus (X_m \otimes H) $
> 类似二项式乘法：将输入分组切割，并作为密钥 H 各阶的系数，与 CBC 的链式特征很相似

![GHASH](ghash.png)

### 3. 计数器操作 $Y=GCTR_K(ICB,X)$

先决条件：底层的128位分组密码算法$CIPH$
输入参数：128位比特流$X$
输出结果：分组$Y$
核心逻辑：

1. 初始计数器块ICB（Initial Counter Block），构造方式为：96位 IV + 31位0 + 1，
   $J_0 = IV \parallel 0^{31} \parallel 1$
2. 计数器递增
    $CB_1=ICB$
    $CB_i=inc_{32}(CB_{i-1})$
3. 根据流密钥进行加密或解密
    $Y_i= X_i \oplus CIPH_k(CB_i)$

![GCTR](gctr.png)

### 4. 认证加密算法 $(C,T)=GCM-AE_K(IV,P,A)$

先决条件：底层的128位分组密码算法$CIPH$；密钥$K$；标签长度$t$
输入参数：初始化向量$IV$；明文$P$；附加认证数据$A$
输出结果：密文$C$；标签$T$
核心逻辑：

1. 调用 GCTR 生成密文C
    $C = GCTR_k(inc_{32}(J_0),P)$
2. 构造验证数据块S，关键数据是 A 和C，还有一些填充位
    $u = len(C) \ mod\ 128;\ v = len(A) \ mod\ 128$
    $S = GHASH_H(A \parallel 0^v \parallel C \parallel 0^u \parallel [len(A)]_{64} \parallel [len(C)]\_{64})$
3. 计算数据块S的哈希值，并存入tag
    $T = MSB_i(GCTR_k(J_0,S))$

![arch](arch.png)

### 5. 认证解密算法 $P=GCM-AD_K(IV,C,A，T)$

先决条件：底层的128位分组密码算法$CIPH$；密钥$K$；标签长度$t$
输入参数：初始化向量$IV$；密文$C$；附加认证数据$A$；标签$T$
输出结果：明文$P$，或者 FAIL
核心逻辑：

1. 调用 GCTR 生成明文P
    $P = GCTR_k(inc_{32}(J_0),C)$
2. 构造验证数据块S，关键数据是 A 和C，还有一些填充位
    $u = len(C) \ mod\ 128;\ v = len(A) \ mod\ 128$
    $S = GHASH_H(A \parallel 0^v \parallel C \parallel 0^u \parallel [len(A)]_{64} \parallel [len(C)]\_{64})$
3. 计算数据块S的哈希值，并存入$T'$
    $T' = MSB_i(GCTR_k(J_0,S))$
4. 如果$T = T'$，验证通过返回明文P；否则返回 FAIL

## 四、技术分析

### 1. 安全性要求

GCM 在具体的安全模型中被证明是机密性安全，但需要同时满足三个前提条件：

- $len(P) <= 2^{39} - 256$
- $len(A) <= 2^{64} -1$
- $1 <= len(IV) <= 2^{64}-1$

对于任何给定的密钥和初始化向量组合，GCM 加密数据 P 的长度不得高于大约 64 GB。
附加身份验证数据 AAD（Additional Authenticated Data）是不需要加密但需要认证的数据，以明文形式与密文一起传递给接收者，通常包含源IP，源端口，目的IP，IV等信息。
IV 通常为 96 位长度。

身份验证的安全性取决于 MAC 的长度。一般来说，标签长度 t 应该在[128，120，112，104，96]中选择。对于某些应用，t 可能是 64 或 32，但必须限制输入的长度数据和密钥的生命周期。

### 2. 硬件加速

GCM 要求对每个加密和验证数据块（128 位）在有限域中进行一次分组密码操作和一次 128 位乘法。分组密码操作很容易流水线化或并行化；乘法运算也很容易流水线化，并且可以通过一些适度的努力进行并行化。

Intel 添加了`PCLMULQDQ`指令，突出了它对 GCM 的使用。
2011 年，SPARC 添加了`XMULX`和`XMULXHI`指令，它们也执行 64 × 64 位无进位乘法。
2015 年，SPARC 添加了`XMPMUL`指令，该指令对更大的值执行 XOR 乘法，高达 2048 × 2048 位的输入值产生 4096 位的结果。

---

## 参考文献

- [NIST SP800-38D](nistspecialpublication800-38d.pdf)
- [Galois/Counter Mode - WiKi](https://en.wikipedia.org/wiki/Galois/Counter_Mode)
- [GCM与CCM的的规格和加解密过程](https://blog.csdn.net/Chahot/article/details/130407149)
- [aes-gcm 在线加密工具](https://const.net.cn/tool/aes/aes-gcm/)
- [GCM 的C语言代码示例](https://github.com/openluopworld/aes_gcm/blob/master/gcm.c)
