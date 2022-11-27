---
title: 基于椭圆曲线的ECC非对称密码系统
date: 2022-11-28 00:31:04
tags:
---

## 附录一：一次性迪菲-赫尔曼密钥交换协议

一次性迪菲-赫尔曼密钥交换协议（One-Pass Diffie-Hellman Key Agreement），定义在 [NISP SP 800-56A](NIST.SP.800-56Ar2.pdf) 的 6.2.2.2 章节，名称为`One-Pass Diffie-Hellman, C(1e,1s,ECC CDH) Scheme`，含义是：

- 密钥交换通过一次性的DH（Diffie-Hellman）协议，C 是指协因子（Cofactor）
- 使用了1个临时密钥，1个静态密钥
- 加密算法是椭圆曲线加密算法 ECC（Elliptic Curve Cryptography）

> DH 协议最初采用 FFC 算法（Finite Field Cryptography，有限域加密），也称为幂等算法，安全强度较低
> Curve 25519 使用蒙哥马利曲线，y<sup>2</sup> = x<sup>3</sup> + 486662x<sup>2</sup> + x
> 该曲线定义在由素数定义的素数场的二次扩展上，使用基点x = 9
> 这个基点的阶数是
> 其构造不受定时攻击的影响，接受任何32字节的字符串作为有效的公钥，且无需验证
> 蒙哥马利曲线（Montgomery curve）是另一种形式的椭圆曲线，方程如下：

![onePassECCCDH](onePassECCCDH.png)

素数域p = 2^{255}-19，也是名字中25519的由来。

- p = 2<sup>255</sup> − 19
- 域空间大小：q = 
- FR =
- 参数：a = 486662
- 参数：b = 1
- 初始值（可选）：SEED = N/A
- 生成元：G = 
- 辅因子：h = 8
- 阶数（群秩）：n = 2<sup>252</sup>+27742317777372353535851937790883648493


对于`Protected Unless Open`的文件保护类型，分析其实现方案是：

### 1. 初始化存储介质的环节

- 安全隔区随机生成一对非对称密钥（静态）
- 静态私钥就是 Class B Key，包裹后的密文**持久化存储**在系统密钥包

### 2.（某个）文件创建的环节

- 安全隔区随机生成一个文件独有密钥 per-file key
- 安全隔区随机生成一对非对称密钥（临时），封装密钥 =（静态私钥，临时公钥），生成 per-file key！
- 安全隔区将（临时公钥，per-file key！）发给操作系统
- 操作系统创建 cnode 元数据，持久化存储 per-file key!，并增加了一个字段存储临时公钥
- 安全隔区将 per-file key 发给 AES 引擎，实现文件内容的加密存储

> 临时私钥从为被使用？好像没有哪个文档提及其用途

### 3.（重新）打开文件的环节

- 操作系统作从文件元数据中获得（临时公钥，per-file key！），并发送给安全隔区
- 安全隔区以 Class B key 作为静态私钥，解封密钥 =（静态私钥，临时公钥），将 per-file key！解封为 per-file key
- 安全隔区将 per-file key 发给 AES 引擎，实现文件内容的加密读写
- 一旦该文件被关闭，安全隔区就在内存中丢弃 per-file key，该文件将无法读写

### 4. 用户重置passcode

理论上应有如下操作：

1. 根据新的 passcode 重新执行 KDF 函数，生成新的 Class B key
2. 找到所有数据保护类型为 Class B 的文件元数据，执行如下循环操作：
    - 解封密钥 =（旧的静态私钥，临时公钥），将 per-file key！恢复为 per-file key
    - 封装密钥 =（新的静态私钥，临时公钥），将 per-file key 封装为新的 per-file key！
    - 操作系统接受新的 per-file key！，并持久化存储在元数据 cnode 的 cprotect 字段
3. 循环结束后，将新的 Class B key 持久化存储在系统钥匙包

> 静态私钥保存在 Class B key，但没有文档提及其用途和持久化存储位置，似乎被丢弃了
> 临时私钥的情况也是类似，而且似乎丢弃了也不影响算法的实现。。。


![onePassECCCDH](onePassECCCDH-2.png)




