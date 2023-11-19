---
title: AES-XTS 磁盘加密模式的技术分析
date: 2023-10-29 20:51:15
tags:
---


AES-XTS 的全称：XEX-based Tweaked-codebook mode with ciphertext Stealing。

- 基于XEX，XOR-ENCRYPT-XOR
- 支持密文窃取算法，ciphertext Stealing
- 可调整的密码本模式，Tweaked-codebook

## 一、背景

AES-XTS 的设计目标是解决磁盘文件的加密存储需求。
ECB 模式存在明文攻击问题，磁盘文件中经常存在内容相同的分组，显然不适用。其他分组密码工作模式也有两个难题无法解决，一是如何存储初始向量IV，二是如何支持随机存取。

CBC、CFB 和 OFB 模式都需要初始向量，因此每个文件都需要额外的空间用于存储 IV，不仅增加磁盘开销，而且破坏了明文和密文在扇区存储上的对应关系，给底层技术实现带来了很大麻烦。

CBC、CFB 和 OFB 模式的分组之间存在依赖关系，不能实现完全的并行计算（CBC、CFB 模式仅支持解密的并行计算，并需要额外读取上一个分组的密文；OFB 模式完全不支持并行计算），因此即使仅仅修改磁盘文件的个别字节，也需要重新加密整个文件，严重影响了处理效率。

相对而言，CTR 模式较为友好，通过提前预处理流密钥，可以支持加密和解密的并行计算。但是，CTR 模式同样需要独立的计数器，无法解决额外存储的问题。

2002年，Moses Liskov，Ronald L.Rivest, David Wagner 首次提出了[可调整的分组密码](https://people.csail.mit.edu/rivest/pubs/LRW02.pdf)，跟传统的分组密码相比，除了密匙和明文这两个输入外，还引入另外一个输入---tweak，即可调整值，其优势在于：

1. 无需更改分组密钥，更改 tweak value 就可以给加密系统提供多变性，tweak value 是公开的，即使被泄露，仍然需要分组密钥才能解密；
2. 完成初始化加密后，tweak value 甚至可以任意修改而不影响密文的解密，也意味着 tweak 修改后不需要重新计算并存储加密数据；
3. tweak value 可以设置为与区块的 index 成对应关系，不再需要存储额外的初始向量，也就避免了明文和密文在扇区上的存储不对应的问题；
4. 各个区块数据的加密解密互相独立（tweak key 各不相同），不存在依赖关系，加密和解密都可以完全并行化。

2007年，IEEE 组织发布了[IEEE std 1619-2007](https://luca-giuzzi.unibs.it/corsi/Support/papers-cryptography/1619-2007-NIST-Submission.pdf) ，提供 XTS-AES 算法用于以数据单元（包括扇区、逻辑磁盘块等）为基础结构的存储设备中静止状态数据的加密。
2010年，美国密码局也接纳了该标准，并命名为[NIST SP800-38E](https://csrc.nist.gov/pubs/sp/800/38/e/final)。

## 二、基本原理

![total](XTS_mode_encryption.png)

### 1. 单一分组的处理方式

#### 加密算法

XTS-AES 算法的公式表示：$C = XTS-AES-blockEnc(Key, P, i, j)$。
输入参数为：

- $Key$：一个加密密钥，均等分为两个部分，key2 用于调整 tweak，key1 用于明文的分组加密
- $P$：一个128位的明文块
- $i$：一个128位的 tweak 值
- $j$：这个块在数据单元中的索引

输出结果是：

- $C$：这个块加密后的密文

![xts](enc.png)

处理逻辑为：
$T = AES-enc(key_2, i) \bigotimes \alpha^j$
$C = AES-enc(key_1, P \bigoplus T) \bigoplus T$

#### 解密算法

XTS-AES 算法的公式表示：$C = XTS-AES-blockDec(Key, C, i, j)$。
输入参数为：

- $Key$：一个加密密钥，均等分为两个部分，key2 用于调整 tweak，key1 用于明文的分组加密
- $C$：这个块加密后的密文
- $i$：一个128位的 tweak 值
- $j$：这个块在数据单元中的索引

输出结果是：

- $P$：一个128位的明文块

![xts](dec.png)

处理逻辑为：
$T = AES-enc(key_2, i) \oplus \alpha^j$
$P = AES-dec(key_1, C \oplus T) \oplus T$

> 解密和加密的处理逻辑保持对称性，首先生成流密码，然后分别进行加密或者解密的计算

#### 有限域的乘法运算

域是群里的特殊群，域有单位元e和逆元，有限域是该域的元素确定，而不是无限。域有这样一个性质：在加法和乘法上具有封闭性。也就是说对域中的元素进行加法或乘法运算后的结果仍然是域中的元素。有一点要注意，域里面的乘法和加法不一定是我们平常使用的乘法和加法。可以把C语言中的与运算和异或运算分别定义成加法和乘法。

GF(M)中M必须为素数，所以用GF( 
 )，它的p必须为素数，才能解决一些非素数的运算。

GF( 
 ),正好是一个字节的比特位数，所以广泛应用于计算机领域。


### 2. 密文窃取的处理方式



### 最后2个分组的加密算法

![xts](enc-last2.png)

if bit-size_of($C_m$) = 0 then do
    $P_{m-1} = XTS-AES-blockDec(Key,C_{m-1},i,m-1)$
    $P_m = empty$

### 最后2个分组的解密算法

![xts](dec-last2.png)

1. tweak使用密钥K2进行AES加密(ECB模式， 不需要iv），结果进行一个有限域数aj的乘运算，得到T
2. T跟明文块P异或得到PP
3. PP再使用K1进行AES加密得到CC
4. CC再和1中的T异或得到最终的密文C

### 3. 实现案例

Mac OS X Lion's FileVault 2、Windows 10's BitLocker 等主流文件系统均已实现XTS-AES 算法，并满足如下技术要求：

- 密⽂文应该与明⽂⼤小相同，从⽽不改变数据的布局
- 数据分组可单独访问，且相互独⽴
- 加密分组⼤小为16字节
- 不使⽤其他元数据（除了数据分组的位置）
- 不同位置 相同明⽂ ==》密⽂不不同
- 符合标准的加解密设备可以相互通⽤

`tweak key = SHA256(128位的 volume master key + 128位的 logical volume family identifier)`

---

## 参考文献

- [系列 - 写给开发人员的实用密码学](https://thiscute.world/posts/practical-cryptography-basics-1/)
- [分组加密工作模式 - Yang‘s blog](http://stuyang.com/blog/6fdd6732f56d/)
- [分组密码与流密码：它们是什么以及它们如何工作](https://www.asiaregister.com/zh/news/fen-zu-mi-ma-yu-liu-mi-ma-ta-men-shi-shen-me-yi-ji-ta-men-ru-he-gong-zuo-2453.htm)
- [分组密码工作模式 - WiKi](https://zh.m.wikipedia.org/zh-hans/%E5%88%86%E7%BB%84%E5%AF%86%E7%A0%81%E5%B7%A5%E4%BD%9C%E6%A8%A1%E5%BC%8F)
- [OS X Disk Util Manual Basic](https://zhuanlan.zhihu.com/p/20279484)
- [FileVault Drive Encryption (FVDE)](https://github.com/libyal/libfvde/blob/main/documentation/FileVault%20Drive%20Encryption%20(FVDE).asciidoc)
- [AES with XTS mode example - IBM](https://www.ibm.com/docs/en/linux-on-systems?topic=examples-aes-xts-mode-example)
- [伽罗瓦域上的乘法](http://blog.foool.net/2013/01/伽罗瓦域上的乘法/)
- [另一种世界观——有限域](https://www.bilibili.com/read/cv2922069/)

### 文档下载

- [分组密码 工作模式建议：NIST SP 800-38A 2001 Edition](nistspecialpublication800-38a.pdf)
- [分组密码 GCM/GMAC 工作模式：NIST SP 800-38D 2001 Edition](nistspecialpublication800-38d.pdf)
- [分组密码 XTS-AESC 工作模式：NIST SP 800-38E 2001 Edition](nistspecialpublication800-38e.pdf)
- [IEEE Std 1619-2007, The XTS-AES Tweakable Block Cipher](1619-2007-NIST-Submission.pdf)
- [现代密码学理论与实践 - 苗付友.pdf](现代密码学理论与实践-苗付友.pdf)

---

1993年，Shimon Even 和 Yishay Mansour 发表了[A Construction of a Cipher from a Single Pseudorandom Permutation](https://link.springer.com/content/pdf/10.1007/s001459900025)

【Apple 平台安全保护    2021 年 5 月】

文件保险箱已打开的内部储存设备
如果没有有效的登录凭证或加密恢复密钥， 即使物理储存设备被移除并连接到其他电脑， 内置 APFS 宗卷仍保持加密状态， 以防止未经授权的访问。
在 macOS 10.15 中， 此类宗卷同时包括系统宗卷和数据宗卷。 
从macOS 11 开始， 系统宗卷通过签名系统宗卷 (SSV) 功能进行保护， 而数据宗卷仍通过加密进行保护。 
搭载Apple 芯片的 Mac 以及搭载 T2 芯片的 Mac 通过构建和管理密钥层级实施内部宗卷加密， 基于芯片内建的硬件加密技术而构建。 此密钥层级的设计旨在同时实现四个目标 ：
• 请求用户密码用于加密
• 保护系统免受针对从 Mac 上移除的储存媒介的直接暴力破解攻击
• 提供擦除内容的快速安全的方法， 即通过删除必要的加密材料
• **让用户无需重新加密整个宗卷即可更改其密码 （同时也会更改用于保护其文件的加密密钥）**

## 算法分析

加密和解密都用T1

$$
\begin{aligned}
    P'&=AES-dec(key_1,C\oplus T_1)\oplus T_1\\\\
    &=AES-dec(key_1, AES-enc(key_1, P \oplus T_1) \oplus T_1 \oplus T_1) \oplus T_1 \\\\
    &=AES-dec(key_1, AES-enc(key_1, P \oplus T_1) \oplus 0) \oplus T_1 \\\\
    &=AES-dec(key_1, AES-enc(key_1, P \oplus T_1)) \oplus T_1 \\\\
    &=P \oplus T_1 \oplus T_1 \\\\
    &=P \oplus 0 \\\\
    &=P
\end{aligned}
$$

换了T2

$$
\begin{aligned}
    P''&=AES-dec(key_1,C\oplus T_2)\oplus T_2\\\\
    &=AES-dec(key_1, AES-enc(key_1, P \oplus T_1) \oplus T_1 \oplus T_2) \oplus T_2 \\\\
    &=AES-dec(key_1, AES-enc(key_1, P \oplus T_1) \oplus T_1 \oplus T_2 \oplus T_2) \\\\
    &=AES-dec(key_1, AES-enc(key_1, P \oplus T_1) \oplus T_1 \oplus 0) \\\\
    &=AES-dec(key_1, AES-enc(key_1, P \oplus T_1) \oplus T_1) \\\\
    &=AES-dec(key_1, AES-enc(key_1, P \oplus T_1 \oplus T_1)) \\\\
    &=AES-dec(key_1, AES-enc(key_1, P \oplus 0)) \\\\
    &=AES-dec(key_1, AES-enc(key_1, P)) \\\\
    &=P
\end{aligned}
$$

[XEX](https://en.wikipedia.org/wiki/Xor%E2%80%93encrypt%E2%80%93xor)

```c
#define GF_128_FDBK     0x87                // 设定本原生成式 = x^4 + x^2 + x + 1，FDBK=feedback
#define AES_BLK_BYTES   16                  // 1 byte = 8 bits, 128位 = 16个字节

void XTS_EncryptSector                      // 5个入参
    (
    AES_Key &k2,                            // tweak 可调整密钥
    AES_Key &k1,                            // ECB 主密钥
    u64b    S,                              // 分组序号，data units number（64 bits）｜sector number
    uint    N,                              // sector size，in bytes
    const   u08b    *pt,                    // 明文，输入
            u08b    *ct                     // 密文，输出
    )
{                                           // 函数主体代码
    uint    i,j;
    u08b    T[AES_BLK_BYTES];               // tweak value， 
    u08b    x[AES_BLK_BYTES];               // 临时工作变量
    u08b    Cin, Cout;                      // LSFR 的携位信息

    assert(N % AES_BLK_BYTES == 0);         // 断言：确认数据块是 16 的整数倍

    // 将分组序号 S 按位导入 T，并调整为小端字节序：123456789a->9a78563412
    for (j = 0; j < AES_BLK_BYTES; j++>) {  
        T[j] = (u08b) (S & 0xFF);           
        S = S >> 8;
    }

    // 通过 ECB 加密，完成 T 的初始化
    AES_ECB_Encrypt(k2, T);                 T

    for(i = 0; i < N; i += AES_BLK_BYTES) {
        // 明文与 T 的第一轮XOR
        for (j = 0; j < AES_BLK_BYTES; j++)
            x[j] = pt[i+j] ^ T[j];          

        // 完成主密钥加密
        AES_ECB_Encrypt(k1, x);             

        // 明文与 T 的第二轮XOR
        for (j = 0; j < AES_BLK_BYTES; j++) 
            ct[i+j] = x[j] ^ T[j];          // 主密钥的输出后，按位 XOR

        // 构造下一轮的T，包含移位和反馈两个步骤
        // 移位步骤：T 的16个字节依次左移1位，末位补0。不能简单的整体左移1位！因为是小端字节序
        Cin = 0;
        for (j = 0; j < AES_BLK_BYTES; j++) {   
            Cout = (T[j] >> 7 ) & 1;            // 第j个字节的溢出位，暂存Cout
            T[j] = ((T[j] << 1) + Cin) & 0xFF;  // 当前字节左移1位，末位填充上个字节的溢出位
            Cin = Cout;                         // 为下个字节准备溢出位
        }
        // 反馈步骤：如果T最高位溢出的比特为1，则最低位字节 XOR (1000 0111)
        if (Cout)                           
            T[0] ^= GF_128_FDBK;            
    }
}

```

---

```c
#define GF_128_FDBK     0x87                // 设定本原生成式 = x^4 + x^2 + x + 1，FDBK=feedback
#define AES_BLK_BYTES   16                  // 1 byte = 8 bits, 128位 = 16个字节

void XTS_EncryptSector                      // 5个入参
    (
    AES_Key &k2,                            // tweak 可调整密钥
    AES_Key &k1,                            // ECB 主密钥
    u64b    S,                              // 扇区sector的序号，最长64位
    uint    N,                              // 扇区sectot的长度，按字节计算
    const   u08b    *pt,                    // 明文，输入
            u08b    *ct                     // 密文，输出
    )
{                                           // 函数主体代码
    uint    i,j;
    u08b    T[AES_BLK_BYTES];               // tweak value， 
    u08b    x[AES_BLK_BYTES];               // 临时工作变量
    u08b    Cin, Cout;                      // LSFR 的携位信息

    assert(N >= AES_BLK_BYTES == 0);         // 断言：确认至少有一个完整分组

    // 将扇区序号作为 i，导入tweak value；注意需调整为小端字节序：123456789a->9a78563412
    for (j = 0; j < AES_BLK_BYTES; j++>) {  
        T[j] = (u08b) (S & 0xFF);           
        S = S >> 8;                         // S 是64位，高8个字节实际填充0；物理限制2^128，建议2^20
    }

    // 基于 tweakl key 加密，完成 T 的初始化
    AES_ECB_Encrypt(k2, T);                 T

    // 每次处理16字节的分组，i 记录当前扇区的处理位置，注意循环结束在倒数第二个分组！
    for(i = 0; i+AES_BLK_BYTES <= N; i += AES_BLK_BYTES) {
        // 明文 pt 和 T 的第一轮XOR
        for (j = 0; j < AES_BLK_BYTES; j++)
            x[j] = pt[i+j] ^ T[j];          

        // 基于主密钥完成加密
        AES_ECB_Encrypt(k1, x);             

        // 明文 pt 和 T 的第二轮XOR，ct 作为密文结果输出
        for (j = 0; j < AES_BLK_BYTES; j++) 
            ct[i+j] = x[j] ^ T[j];         

        // 基于GF(2^128)乘法，LSFR 为下一循环构造 T，包含移位和反馈两个步骤
        // 移位步骤：T 的16个字节依次左移1位，末位补0。不能简单的整体左移1位！因为是小端字节序
        Cin = 0;
        for (j = 0; j < AES_BLK_BYTES; j++) {   
            Cout = (T[j] >> 7 ) & 1;            // 第j个字节的最高位溢出，暂存Cout
            T[j] = ((T[j] << 1) + Cin) & 0xFF;  // 当前字节左移1位，末位填充上个字节的溢出位
            Cin = Cout;                         // 为下个字节准备溢出位
        }
        // 反馈步骤：如果T最高位溢出的比特为1，则最低位字节 XOR (1000 0111)
        if (Cout)                           
            T[0] ^= GF_128_FDBK;                // 本原生成式的最高位x^4，T[0]就够用
    }

    // 如果 i == N，说明没有不完整分组，无需处理；否则，对最后一个不完整扇区进行密文窃取
    if (i < N) {   
        // 明文 pt 和 T 的第一轮XOR                         
        for (j=0; i+j<N; j++) {
            x[j] = pt[i+j] ^ T[j];              // x[0..z-1]：仅处理明文的剩余 z 个字节 
            ct[i+j] = ct[i+j-AES_BLK_BYTES];    // 保存上一分组密文的前 z 个字节，并追加到密文输出的尾部
        }
        for (; j<AES_BLK_BYTES; j++)
            x[j] = ct[i+j-AES_BLK_BYTES];       // x[z,15]：窃取上一轮密文的后 16-z 个字节

        // 基于主密钥完成加密
        AES_ECB_Encrypt(k1, x);                 

        // 填充窃取数据的明文 pt 和 T 的第二轮XOR，ct 作为密文结果输出
        for (j=0; j<AES_BLK_BYTES; j++)
            ct[i+j-AES_BLK_BYTES] =x[j] ^ T[j]; 
    }                                
}

```


---

状态的概念：
一个LFSR寄存器中当前存储的序列被称为一个状态。在LFSR输出一位，由反馈函数补充一位后，LFSR就移动到了下一个状态。

为了能够产生足够安全的密钥，我们通常要求LFSR的周期能够足够大。上文提到，一个n级LFSR最多只能遍历  个状态，这也就是说，一个n级LFSR的最大周期就是  。

我们把周期为  的LFSR所生成的序列称为m序列。

6、本原多项式：

m序列LFSR反馈函数对应的特征多项式被称为本原多项式。


start_state = 1 << 15 | 1
lfsr = start_state
period = 0

while True:
    #taps: 16 15 13 4; feedback polynomial: x^16 + x^15 + x^13 + x^4 + 1
    bit = (lfsr ^ (lfsr >> 1) ^ (lfsr >> 3) ^ (lfsr >> 12)) & 1
    lfsr = (lfsr >> 1) | (bit << 15)
    period += 1
    print(period,':', lfsr)
    if (lfsr == start_state):
        print('END')
        break

---

域中不可约多项式（primitive polynomial）是不能够进行因子分解的多项式，本原多项式是一种特殊的不可约多项式。当一个域上的本原多项式确定了，这个域上的运算也就确定了，本原多项式一般通过查表可得，同一个域往往有多个本原多项式。通过将域中的元素化为多项式的形式，可以将域上的乘法运算转化为普通的多项式乘法模以本原多项式的计算。比如g(x) = x3+x+1 是GF(23)上的本原多项式，那么GF(23) 域上的元素3*7 可以转化为多项式乘法：
3*7(in GF(23)) → (x+1)*(x2+x+1) mod g(x) = x3+1 mod x3+x+1 = x → 3*7 = 2 (in GF(23))
需要注意的是，系数为2 的整数倍会被约去。


无论是普通计算还是伽罗瓦域上运算，乘二计算是一种非常特殊的运算。普通计算在计算机上通过向高位的移位计算即可实现，伽罗瓦域上乘二也不复杂，一次移位和一次异或即可，并且2 是GF(28) 中的生成元。

从多项式的角度来看，伽罗瓦域上乘二对应的是一个多项式乘以x，如果这个多项式最高指数没有超过本原多项式最高指数，那么相当于一次普通计算的乘二计算，如果结果最高指数等于本原多项式最高指数，那么需要将除去本原多项式最高项的其他项和结果进行异或，在硬件中是这样实现的（g(x) = x8+x4+x3+x2+1）：
x7  ← x6 ← x5  ← x4 ← x3 ← x2  ← x1  ← x0
↓_____________↑___↑___↑_________↑
这里x0到x7 分别代表域中一个数的比特位，每次乘二除最高位不移位，其余各位向高位移一位，最高位和指定位进行异或（由本原多项式决定）。不难知道最高位异或的那几位对应着本原多项式系数为1 的几项。
用C 语言可以写成：

```c
    uint8_t c, cc;
    cc = (c<<1) ^ ((c & 0x80) ? 0x1d : 0)
```

c&0x80 可以判断数c 最高位是否为1，如果为1 则异或ox1d。