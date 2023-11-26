---
title: AES-XTS 磁盘加密模式的技术分析
date: 2023-10-29 20:51:15
tags:
---

AES-XTS 的设计目标是解决磁盘文件的加密存储需求。
ECB 模式存在明文攻击问题，磁盘文件中经常存在内容相同的分组，显然不适用。其他分组密码工作模式也有两个难题无法解决，一是如何存储初始向量IV，二是如何支持随机存取。

CBC、CFB 和 OFB 模式都需要初始向量，因此每个文件都需要额外的空间用于存储 IV，不仅增加磁盘开销，而且破坏了明文和密文在扇区存储上的对应关系，给底层技术实现带来了很大麻烦。

CBC、CFB 和 OFB 模式的分组之间存在依赖关系，不能实现完全的并行计算（CBC、CFB 模式仅支持解密的并行计算，并需要额外读取上一个分组的密文；OFB 模式完全不支持并行计算），因此即使仅仅修改磁盘文件的个别字节，也需要重新加密整个文件，严重影响了处理效率。

相对而言，CTR 模式较为友好，通过提前预处理流密钥，可以支持加密和解密的并行计算。但是，CTR 模式同样需要独立的计数器，无法解决额外存储的问题。

## 一、概述

AES-XTS 的全称：XEX-based Tweaked-codebook mode with ciphertext Stealing。

- 基于XEX，XOR-ENCRYPT-XOR
- 支持密文窃取算法，ciphertext Stealing
- 可调整的密码本模式，Tweaked-codebook

白化（Whitening）是 AES-XTS 的重要技术基础，最早出现是 Ronald L.Rivest 发明的 DES-X。
DES 通常使用的密钥是56位，为了增加迭代分组加密安全性，DES-X 使用了两个64位的密钥，在第一轮加密和最后一轮对密钥和明文进行异或操作，攻击者若不知道密钥，则无法进行第一个加密解密操作，这样就在不改变算法的前提下，即可增加密钥的有效长度，以增强暴力破解的复杂度。
> Ronald L.Rivest 就是 RSA 中的 R ！美国麻省理工学院教授，2002年图灵奖得主之一。

2002年，Moses Liskov，Ronald L.Rivest, David Wagner 首次提出了[可调整的分组密码](https://people.csail.mit.edu/rivest/pubs/LRW02.pdf)，跟传统的分组密码相比，除了密匙和明文这两个输入外，还引入一个新输入---可调整值 tweak，其优势在于：

1. 在不更改分组密钥的情况下，仅仅改变 tweak value 就可以给加密系统提供多变性，tweak value 是公开的，即使被泄露，仍然需要分组密钥才能解密；
2. tweak value 可以设置为与区块的 index 成对应关系，不再需要存储额外的初始向量，也就避免了明文和密文在扇区上的存储不对应的问题；
3. 完成初始化加密后，tweak value 甚至可以任意修改而不影响密文的解密，也意味着 tweak 修改后不需要重新计算并存储加密数据；
4. 各个区块数据的加密解密互相独立（tweak key 各不相同），不存在依赖关系，加密和解密都可以完全并行化。

2007年，IEEE 组织发布了[IEEE std 1619-2007](https://luca-giuzzi.unibs.it/corsi/Support/papers-cryptography/1619-2007-NIST-Submission.pdf) ，提供 XTS-AES 算法用于以数据单元（包括扇区、逻辑磁盘块等）为基础结构的存储设备中静止状态数据的加密，SISWG（存储安全工作组，Security in Storage Working Group）负责维护该标准。

![total](XTS_mode_encryption.png)

2010年，美国密码局也接纳了该标准，并命名为[NIST SP800-38E](https://csrc.nist.gov/pubs/sp/800/38/e/final)。

## 二、基本流程

### 1. 单一分组的处理方式

![vs](enc-dec.png)

#### 加密算法：$C = XTS-AES-blockEnc(Key, P, i, j)$

输入参数为：

- $Key$：一个加密密钥，均等分为两个部分，key2 就是 tweak key，key1 用于明文的分组加密
- $P$：一个128位的明文块，也就是16个字节
- $i$：一个128位的 tweak value，通常是磁盘扇区的序号
- $j$：该明文块在该数据单元（扇区）中的分组序列号，以16个字节为单位

输出结果是：

- $C$：这个块加密后的密文

处理逻辑为：
$T = AES-enc(key_2, i) \bigotimes \alpha^j$
$C = AES-enc(key_1, P \bigoplus T) \bigoplus T$

#### 解密算法：$C = XTS-AES-blockDec(Key, C, i, j)$

输入参数、输出结果的结构与加密算法保持一致，只是 C 和 P 换了位置。

处理逻辑为：
$T = AES-enc(key_2, i) \bigotimes \alpha^j$
$P = AES-dec(key_1, C \bigoplus T) \bigoplus T$

### 2. 密文窃取的处理方式

![2](enc-dec-last2.png)

#### 加密算法

- 明文$P_{m-1}$执行 AES-enc，密文结果 $C$ 的前 i 个字节作为$C_{m-1}$
- （明文$P_m$的不完整分组 + **窃取**$C$的剩余字节），执行 AES-enc 得到$C_m$

#### 解密算法

- 对密文$C_{m-1}$执行 AES-dec，明文结果 P 的前 i 个字节作为$P_m$
- （密文$C_m$的不完整分组 + **窃取**$P$的剩余字节），执行 AES-dec 得到$P_{m-1}$

通过密文窃取方式，AES-XTS 保持了明文和密文长度的一致性，不会造成存储空间的浪费，代价是需要缓存最后2个数据分组。

## 三、代码示例

AES-XTS 的硬件实现很方便，有限域的乘法就是一个 LSFR（线性移位反馈寄存器）。

```c
#define GF_128_FDBK     0x87                // 设定本原生成式 = x^4 + x^2 + x + 1，FDBK=feedback
#define AES_BLK_BYTES   16                  // 1 byte = 8 bits, 128位 = 16个字节

void XTS_EncryptSector( AES_Key &k2,       // tweak 可调整密钥 
                        AES_Key &k1,       // ECB 主密钥
                        u64b    S,         // 扇区sector的序号，最长64位
                        uint    N,         // 扇区sectot的长度，按字节计算
                        const   u08b *pt,  // 明文，输入，const声明防止被修改
                        u08b    *ct )      // 密文，输出，地址传参
{                                          
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
        for (; j<AES_BLK_BYTES; j++)            // 注意！j 开始于上个循环的结束位置 m-1 ！
            x[j] = ct[i+j-AES_BLK_BYTES];       // x[z,15]：窃取上一轮密文的后 16-z 个字节

        // 基于主密钥完成加密
        AES_ECB_Encrypt(k1, x);                 

        // 填充窃取数据的明文 pt 和 T 的第二轮XOR，ct 作为密文结果输出
        for (j=0; j<AES_BLK_BYTES; j++)
            ct[i+j-AES_BLK_BYTES] =x[j] ^ T[j]; 
    }                                
}
```

## 四、后续演进

### 1. P1619.1

IEEE 1619 定义的 AES-XTS 仅提供机密性保护的能力，不包含完整性保护和身份认证的能力。
AES-XTS 可能被攻击者改写数据而不表现出异常（选择密文攻击），也无法抵御流量分析和重放攻击，请参见 [Compare Blockmode CBC (with diffuser) against XTS](https://crypto.stackexchange.com/questions/5587/what-is-the-advantage-of-xts-over-cbc-mode-with-diffuser)。

P1619.1 定义了若干附加身份认证的加密算法，用于支持长度扩展的存储设备，包括：

- Counter mode with CBC-MAC (CCM)
- Galois/Counter Mode (GCM)
- Cipher Block Chaining (CBC) with HMAC-Secure Hash Algorithm
- XTS-HMAC-Secure Hash Algorithm

### 2. P1619.2

AES-XTS 也被称为窄块加密算法（Narrow-block encryption），其特点是一个扇区被分成多个分组进行加密处理，例如 AES-128 就是 16 个字节，突出优势是硬件实现的效率高，但也存在某些安全隐患，例如较小的分组长度为数据修改攻击提供了便利条件。

对应的，宽块加密算法（wide-block encryption）以一个完整的扇区为单位（通常为 512 个字节）进行加密处理，P1619.2 为此定了若干算法：

- XCB：[the Extended Codebook (XCB) Mode of Operation](https://eprint.iacr.org/2004/278.pdf)
- EME2：Encrypt Mix Encrypt V2 Advanced Encryption Standard

### 3. P1619.3

定义了一套密钥管理的基础架构，包括体系结构、命名空间、操作、消息传递和传输。

## 五、产品实现

### 1. Mac OS X Lion's FileVault 2

Apple 开发的 FileVault 基于 XTS-AES 算法实现，是一个磁盘级加密产品（不是文件级加密），满足如下技术要求：

- 密⽂文应该与明⽂⼤小相同，从⽽不改变数据的布局
- 数据分组可单独访问，且相互独⽴
- 加密分组⼤小为16字节
- 不使⽤其他元数据（除了数据分组的位置）
- 不同位置 相同明⽂ ==》密⽂不不同
- 符合标准的加解密设备可以相互通⽤

Apple 安全白皮书中有一些关于 AES-XTS 的描述，例如：

- Mac 电脑会提供文件保险箱，这是一项内建加密功能，用于保护所有静态数据安全。文件保险箱使用 AES-XTS 数据加密算法保护内部和可移除储存设备上的完整宗卷。
- 文件保险箱 FileValut 构建的密钥层级需要满足四个目标，其中之一就是：**让用户无需重新加密整个宗卷即可更改其密码 （同时也会更改用于保护其文件的加密密钥）**。
- 每次在数据宗卷中创建文件时，数据保护都会创建一个新的 256 位密钥 （文件独有密钥），并将其提供给硬件 AES 引擎， 此引擎会使用该密钥在文件写入闪存时对其进行加密。
- 采用 A9 到 A13、 S5 和 S6 的每一代硬件在 XTS 模式中使用 AES-128， 其中 256 位文件独有密钥会被拆分， 以提供一个 128 位tweak 密钥和一个 128 位 cipher 密钥。
- 搭载 A14 和 M1 的设备上，加密模式升级为 AES-256，密钥派生算法基于 NIST Special Publication 800-108。

根据[FileVault Drive Encryption (FVDE)](https://github.com/libyal/libfvde/blob/main/documentation/FileVault%20Drive%20Encryption%20(FVDE).asciidoc)的描述，其AES-XTS协议的实现方式是：

- primary key（key1）：源自于 Volume Master Key (VMK)，128位
- tweak key（key2）: 源自于 logical volume family identifier，128位
- tweak value：源自于磁盘扇区序号，以 512 字节为单位

执行命令`diskutil coreStorage info`可以查看 logical volume family identifier，或者检索系统变量`com.apple.corestorage.lv.familyUUID`，这充分体现了 twaek key 的可公开性。

### 2. Windows Vista BitLocker

微软开发的 Windows Vista BitLocker 使用的 AES-CBC+Elephant diffuser 模式也采用宽块加密模式，加密粒度是一个扇区，工作原理如下：

1. $K = K_1 ｜ K_2$
2. $PP = P \oplus K_1$
3. PP 经过 2 个 diffuser 算法的作用后，使用 CBC 模式进行加密。
    其中，每个扇区的 $IV = E_{K_2}(e(sector))$
    sector为扇区号，$e()$将扇区号映射为一个唯一的 16 字节的 value。

微软已经证明了使用扩散体(diffuser)后的 CBC 比原来的 CBC 更难攻击，

## 六、技术分析

做个简要的总结分析：

1. tweak key 实际上是一种流密码，解密和加密的算法是同质的；而用于 AES 的主密钥的加密和解密算法是互逆的。
2. AES-XTS 采用窄块加密模式，一个 512 字节的标准扇区将分为 32 个 AES-128 的分组。
3. T 的初始化基于可公开的 $key_2$ 和 扇区序号 i，为文件和扇区级别的加密提供了随机数；
    后续每个分组的 T 密钥将基于 LFSR 的 j 阶乘法，为扇区内部的分组加密提供了随机数。
    通过 i 和 j 的组合使用，既提供了随机因子对抗明文攻击，又不需要额外的 IV 存储。
4. T 密钥是基于$GF(2^{128})$的 j 阶乘法，即确保提供伪随机性，又充分利用有限域的元素空间。
5. 除了密文窃取方式需要缓存最后 2 个分组，AES-XTS 的分组之间不存在依赖关系，可以很好支持并行处理。

还有一些遗留问题：

### 1. AES-XTS 如何加密小文件？

IEEE 1619 明确指出 AES-XTS 不支持小于一个分组的明文长度，如 AES-128-XTS 加密文件不得少于16个字节，那么小文件如何加密呢？
一种可能的方案是，采用类似 PKCS#7 的填充技术（文心一言的回答！），但结果是明文和密文的长度不一致，不利于磁盘管理。
另一种方案是，改用 ECB 工作模式，但解决明文攻击的问题如何解决呢？

### 2. 如果更改 tweak key 会发生什么？

tweak key 基于 XOR-ENCRYPT-XOR 模式，更改密钥时是否需要更新已存储的密文呢？
以下理论推演是否成立？

> 用 T1 加密，解密时换了 T2
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

---

## 附录：伽罗瓦域的乘法

### 有限域

从群论的角度看，域是特殊的群，域有单位元e和逆元，而且加法和乘法上具有封闭性，即：对域中的元素进行加法或乘法运算后的结果仍然是域中的元素。
> 注意！域上的乘法和加法不一定是我们平常使用的乘法和加法。考虑到二进制进位的影响，可以把 C 语言中的 AND 运算和 XOR 运算分别定义成加法和乘法。

如果某个域的元素数量确定（而不是无限），则称为有限域，又叫伽罗瓦域(Galois field)，记作GF(p)，该命名是为纪念法国数学家 Evariste Galois。
有限域GF(p)包含 p 个元素（p必须是质数），分别是 0 到 p-1，例如GF(7)：{0,1,2,3,4,5,6}。

很多情况下 GF(p) 是不够用的，比如在图像像素的色彩空间一般为RGB(红绿蓝)色彩空间，每种颜色的取值范围都是0~255，我们需要另外一种有限域，它应该有$2^8$个元素，记为$GF(2^8)$。

### 本原多项式

域中不可约多项式（primitive polynomial）是不能够进行因子分解的多项式，本原多项式是一种特殊的不可约多项式。当一个域上的本原多项式确定了，这个域上的运算也就确定了，本原多项式一般通过查表可得，同一个域往往有多个本原多项式。

对于$GF(p^n)$来说，通常的加法运算与乘法运算往往不能构成一个域的，所以我们需要引进新的加法运算与乘法运算，即多项式乘法。通过将域中的元素化为多项式的形式，可以将域上的乘法运算转化为普通的多项式乘法模以本原多项式的计算。

比如：$g(x) = x^3+x+1$是$GF(2^3)$上的本原多项式，则域上元素$3 \bigotimes 7$可以转化为多项式乘法：
$(x+1) * (x^2+x+1) $ mod g(x) = ($x^3+1$) mod($x^3+x+1$) = x
也就是 $3 \bigotimes 7 = 2$。需要注意的是，系数为 2 的整数倍会被约去。

### LSFR

无论是普通计算还是伽罗瓦域上运算，乘二计算是一种非常特殊的运算。普通计算在计算机上通过向高位的移位计算即可实现，伽罗瓦域上乘二也不复杂，一次移位和一次异或即可，并且2 是GF(28) 中的生成元。

从多项式的角度来看，伽罗瓦域上乘二对应的是一个多项式乘以x，如果这个多项式最高指数没有超过本原多项式最高指数，那么相当于一次普通计算的乘二计算，如果结果最高指数等于本原多项式最高指数，那么需要将除去本原多项式最高项的其他项和结果进行异或，在硬件中要实现：$g(x) = x^8+x^4+x^3+x^2+1$，即：
x7  ← x6 ← x5  ← x4 ← x3 ← x2  ← x1  ← x0
↓________________↑___↑___↑_________↑
这里x0到x7 分别代表域中一个数的比特位，每次乘二除最高位不移位，其余各位向高位移一位，最高位和指定位进行异或（由本原多项式决定）。不难知道最高位异或的那几位对应着本原多项式系数为1 的几项。

用C 语言可以写成：

```c
    uint8_t c, cc;
    cc = (c<<1) ^ ((c & 0x80) ? 0x1d : 0)
```

c&0x80 可以判断数c 最高位是否为1，如果为1 则异或0x1d。

---

## 参考文献

- [系列 - 写给开发人员的实用密码学](https://thiscute.world/posts/practical-cryptography-basics-1/)
- [XTS-AES模式主要是解决什么问题，是怎样解决的?](https://www.zhihu.com/question/26452995)
- [XEX - Wiki](https://en.wikipedia.org/wiki/Xor%E2%80%93encrypt%E2%80%93xor)
- [A Construction of a Cipher from a Single Pseudorandom Permutation](https://link.springer.com/content/pdf/10.1007/s001459900025)
- [FileVault Drive Encryption (FVDE)](https://github.com/libyal/libfvde/blob/main/documentation/FileVault%20Drive%20Encryption%20(FVDE).asciidoc)
- [XTS-AES 参考代码实现 - Github](https://github.com/heisencoder/XTS-AES)
- [OS X Disk Util Manual Basic](https://zhuanlan.zhihu.com/p/20279484)
- [AES with XTS mode example - IBM](https://www.ibm.com/docs/en/linux-on-systems?topic=examples-aes-xts-mode-example)
- [伽罗瓦域上的乘法](http://blog.foool.net/2013/01/伽罗瓦域上的乘法/)
- [另一种世界观——有限域](https://www.bilibili.com/read/cv2922069/)
- [线性反馈移位寄存器（LFSR）](https://www.cnblogs.com/weijianlong/p/11947741.html)
- [磁盘加密模式分析](磁盘加密模式分析.pdf)
- [Tweaking Tweakable AES XTS Mode](https://crossbowerbt.github.io/xts_mode_tweaking.html)