---
title: 密钥派生函数（KDF）概述
date: 2022-10-18 11:45:34
tags:
---

## 一、什么是 KDF

KDF（Key Derivation Function）的最初用途是密钥派生，即从秘密密码或密码短语生成密钥。

### 1、密钥拉伸（Key Stretching）

在系统要求用户设置密码时，我们习惯用4-6位的数字组合，或者字母、数字和特殊符号的某种组合，这种密码强度肯定无法抵御暴力破解、字典攻击、彩虹表攻击等技术手段，密钥派生函数由此应运而生。

KDF 最基础的用途是将密码和其他弱密钥材料来源转换为强密钥，文绉绉地说，就是采用具有低熵（安全性或随机性）的密钥，并将其扩展为更安全的更长密钥。

```js
derivedKey = keyDerivationFunction(originalKey, salt, difficulty)
```

密钥派生函数接受一个密码（或其他弱密钥材料）作为输入，通过一个特殊函数运行它，然后输出安全密钥材料，关键点是增加了一个随机数作为加密因子，这个随机数被称为**盐（Salt）**，针对预计算攻击或rainbow表的随机数据，而`difficulty` 是难度系数的标记，例如迭代计算的次数。

![KDF](key-derivation-function.png)
以Apple数据保护技术为例，就是将4-6位的用户锁屏密码`Passcode`转化为256位的密文`Passcode Key`。

### 2、密钥分离（Key Separation）

KDF允许从单一的密钥材料（主密钥）生成多个不同用途的子密钥，方法是通过使用不同的 salt 随机数，这也被称为密钥多样化（Key Diversification）。

这种方式可以防止获得派生密钥的攻击者学习关于输入秘密值或任何其他派生密钥的有用信息，子密钥可以控制业务的某个部分，但只有主密钥具有完全控制权。例如，Apple 的 Secure Enclave 就以 UID 为根，派生出用于保护 Class Key 的`Key 0x835`和 用于保护 EMF Key 的`Key 0x89B`。

``` js
Key 0x835 = KDF(UID, bytes("01010101010101010101010101010101"))
Key 0x836 = KDF(UID, bytes("00E5A0E6526FAE66C5C1C6D4F16D6180"))
Key 0x838 = KDF(UID, bytes("8C8318A27D7F030717D2B8FC5514F8E1"))
Key 0x89B = KDF(UID, bytes("183e99676bb03c546fa468f51c0cbd49"))
```

### 3、密钥强化（Key Strengthening）

密钥强化同样使用随机盐扩展密钥，但随后**删除该盐**，这使得生成的密钥更强壮，但也意味着后续无法直接验证，**即使合法用户也必须进行暴力破解验证**。
但是，由于合法用户掌握了passcode等部分信息，暴力破解难度远远低于非法用户，但仍然需要消耗大量算力，因此实际应用中并不多见。

## 二、KDF Vs Hash

理论上，KDF 可以采用 3DES 或 AES 等对称算法，好处是可以通过密钥恢复原文，但密钥保管是个大麻烦，一旦泄露所有密码就全部暴露，因此在实际应用中主要采用哈希算法。

一个理想的加密哈希函数，应当具有如下属性：

- 快速：计算速度要足够快
- 确定性：对同样的输入，应该总是产生同样的输出
- 难以分析：对输入的任何微小改动，都应该使输出完全发生变化
- 不可逆：从其哈希值逆向演算出输入值应该是不可行的。这意味着暴力破解是唯一方法
- 无碰撞：找到具有相同哈希值的两条不同消息应该非常困难（或几乎不可能）

这些特点非常适合密钥派生的业务场景，可以确保即使密码文件本身被泄露也能保护密码，同时又能方便地验证用户密码，代价是要同时保存每个用户的密文和随机数。

![KDF Vs Hash](KDF.png)
可以证明，所有基于哈希的 KDF 都是安全的哈希函数，但并非所有哈希函数都是基于哈希的 KDF。
尽管高吞吐量是通用哈希函数的理想属性，但在密码安全应用程序中则相反，防御暴力破解是主要关注点，因此 KDF 要求的是**故意缓慢的哈希算法**。

KDF 计算速度的"慢”是相对而言的，对于普通用户而言，KDF 通常只需要在登录时被执行一次，因此慢这么一点点完全可以接受，而且用户也完全有足够的资源执行这个 KDF 函数。 但是如果一个黑客想要通过 Hash 碰撞来猜测出用户的密码，那它就必须执行海量的 KDF 计算，这个时候 KDF 的威力就显现出来了 —— 黑客将需要提供海量的 CPU/GPU 计算资源、海量的内存资源才能完成目标，而这显然得不偿失，这样 KDF 就确保了用户密码的安全性。

提升 Hash 算法的碰撞难度，主要从以下三个方向入手：

- 时间复杂度：对应 CPU/GPU 计算资源
- 空间复杂度：对应 Memory 内存资源
- 并行维度：使用无法分解的算法，锁定只允许单线程运算

## 三、主流算法

### 1. crypt

第一个基于密码的密钥派生函数被称为“ crypt ”（或在其手册页之后的“crypt(3)” ），由 Robert Morris 在 1978 年发明。
crypt 算法将加密一个常数（零），使用用户密码的前 8 个字符作为密钥，通过执行修改后的DES加密算法的 25 次迭代（其中使用从实时计算机时钟读取的 12 位数字来干扰计算），生成的 64 位数字被编码为 11 个可打印字符，然后存储在Unix密码文件中。
虽然这在当时是一个巨大的进步，但自 PDP-11 以来处理器速度的提高使得针对 crypt 的暴力攻击成为可能，并且 12 位盐的加密强度不足。此外，crypt 函数的设计还将用户密码限制为 8 个字符，这就限制了密钥空间。

### 2. PBKDF2（Password-Based Key Derivation Function）

基本原理是通过一个伪随机函数（例如 HMAC 函数），把明文和一个随机盐作为输入参数，然后重复进行运算，并最终产生密钥。

```js
DK = PBKDF2(PRF, password, salt, c, dk_len)
```

其中，PRF 表示使用何种哈希算法（通常是 SHA1 或者 SHA256），password 是人类可读密码，salt 是随机生成的盐（一般不能少于8字节），c 是迭代次数（至少1000次），dk_len 是最终生成的 key 的长度，也就是加密算法的块大小。
![PBKDF2](PBKDF2.png)

该算法的优点是标准化，技术实现容易，而且导出密钥的长度本质上是没有限制的，但是其最大有效搜索空间受限于基本伪随机函数的结构。
典型应用是 Wi-Fi 的 WPA2 密码保护协议就是采用这个算法，即：
`DK = PBKDF2(HMAC−SHA1, passphrase, ssid, 4096, 256)`

> 最近几年比特币挖矿的发展，让大家看到专有硬件、GPU 在对付大规模哈希时的威力。像 PBKDF2 这样简单使用 SHA256，看起来已经不太保险了

### 3. Bcrypt（Blowfish crypt）

Blowfish（河豚）是一个对称密钥加密的分组密码算法，由布鲁斯·施奈尔于 1993 年设计用于替代陈旧的 DES 算法，强调完全开源，无须授权即可使用。
1999年，Niels Provos 和 David Mazières 改进了 Blowfish 算法，通过增加每次迭代的计算开销，达到提升破解难度的目标，并成为 OpenBSD 和许多Linux 发行版（如SUSE Linux）的默认密码哈希算法。

Bcrypt的工作原理，就是对字符串`OrpheanBeholderScryDoubt`进行64次 blowfish 加密得到的结果，而用户密码就是该字符串的加密因子之一。

``` java
Function bcrypt
   Input:
      cost:     Number (4..31)                      log2(Iterations). e.g. 12 ==> 212 = 4,096 iterations
      salt:     array of Bytes (16 bytes)           random salt
      password: array of Bytes (1..72 bytes)        UTF-8 encoded password
   Output: 
      hash:     array of Bytes (24 bytes)

   // 初始化方法，用于生成 K 数组和 S-box 空间，此步骤非常耗时
   //P: array of 18 subkeys (UInt32[18])
   //S: Four substitution boxes (S-boxes), S0...S3. Each S-box is 1,024 bytes (UInt32[256])
   P, S <- EksBlowfishSetup(cost, salt, password)   

   // 基于 24 位初始向量，使用 Blowfish 算法的 ECB 模式，进行 64 轮的迭代加密
   ctext <- "OrpheanBeholderScryDoubt"  //24 bytes ==> three 64-bit blocks
   repeat (64)
      ctext <-  EncryptECB(P, S, ctext) //encrypt using standard Blowfish in ECB mode

   //24-byte ctext is resulting password hash
   return Concatenate(cost, salt, ctext)
```

BCrypt算法将 salt 随机并混入最终加密后的密码，形成一个 60 位的 bcrypt hash 密文，因此验证时无需单独提供之前的 salt。

``` shell
$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy
\__/\/ \____________________/\_____________________________/
 Alg Cost      Salt                        Hash
```

该表示中 `$` 是各个字段的分隔符，包括：

- 2a：  表示的hash算法的唯一标志。这里表示的是bcrypt算法。
- 10：  表示的是代价因子，这里是2的10次方，也就是1024轮。
- N9qo8uLOickgx2ZMRZoMye ：             是16个字节的 salt 经过 base64 编码得到的 22 长度的字符
- IjZAgcfl7p92ldGxad68LJZdL17lhWy：     是24个字节的 hash，经过 bash64 的编码得到的 31 长度的字符

> Blowfish算法由于分组长度太小已被认为不安全，替代方案是使用 Twofish 密码，有128、192、256位三种密钥长度可供选择，块大小为128位

### 4. Scrypt

比特币最最被人诟病的就是它使用的POW算法，谁的算力高，谁就可以挖矿。传统的基于 CPU 计算的算法逐渐被一些特制的 ASIC / FPGA 处理器打败，这些专用处理器不做别的，就是破解你的密码或者进行哈希运算，为此科学家们发明了很多其他的算法，比如需要占用大量内存的算法，因为内存不像CPU可以疯狂提速，所以限制了很多暴力破解的场景。

Scrypt 是一种密码衍生算法，它是由 Colin Percival 创建的，特点是根据初始化的主密码来生成系列的衍生密码，2016年已列入 RFC 7914 标准。
Scrypt 内部用的还是 PBKDF2 （Salsa20算法），但会长时间地维护一组非常大的伪随机数序列，用于后续的 key 生成过程中，其算法表达式为：

``` js
DK = Scrypt(salt, dk_len, n, r, p)
```

其中的 salt 是一段随机的盐，dk_len 是输出的哈希值的长度。n 是 CPU/Memory 开销值，越高的开销值，计算就越困难。r 表示块大小，p 表示并行度。
在高敏感的场景，推荐使用参数为 `n = 2 ** 20, r = 8`，此时需要消耗 1 GB内存。

``` java
Function scrypt
   Inputs: This algorithm includes the following parameters:
      Passphrase:                Bytes    string of characters to be hashed
      Salt:                      Bytes    string of random characters that modifies the hash to protect against Rainbow table attacks
      CostFactor (N):            Integer  CPU/memory cost parameter – Must be a power of 2 (e.g. 1024)
      BlockSizeFactor (r):       Integer  blocksize parameter, which fine-tunes sequential memory read size and performance. (8 is commonly used)
      ParallelizationFactor (p): Integer  Parallelization parameter. (1 .. 232-1 * hLen/MFlen)
      DesiredKeyLen (dkLen):     Integer  Desired key length in bytes (Intended output length in octets of the derived key; a positive integer satisfying dkLen ≤ (232− 1) * hLen.)
      hLen:                      Integer  The length in octets of the hash function (32 for SHA256).
      MFlen:                     Integer  The length in octets of the output of the mixing function (SMix below). Defined as r * 128 in RFC7914.
   Output:
      DerivedKey:                Bytes    array of bytes, DesiredKeyLen long

   Step 1. Generate expensive salt
   blockSize ← 128*BlockSizeFactor  // Length (in bytes) of the SMix mixing function output (e.g. 128*8 = 1024 bytes)

   Use PBKDF2 to generate initial 128*BlockSizeFactor*p bytes of data (e.g. 128*8*3 = 3072 bytes)
   Treat the result as an array of p elements, each entry being blocksize bytes (e.g. 3 elements, each 1024 bytes)
   [B0...Bp−1] ← PBKDF2HMAC-SHA256(Passphrase, Salt, 1, blockSize*ParallelizationFactor)

   Mix each block in B Costfactor times using ROMix function (each block can be mixed in parallel)
   for i ← 0 to p-1 do
      Bi ← ROMix(Bi, CostFactor)

   All the elements of B is our new "expensive" salt
   expensiveSalt ← B0∥B1∥B2∥ ... ∥Bp-1  // where ∥ is concatenation
 
   Step 2. Use PBKDF2 to generate the desired number of bytes, but using the expensive salt we just generated
   return PBKDF2HMAC-SHA256(Passphrase, expensiveSalt, 1, DesiredKeyLen);

Function ROMix(Block, Iterations)

Create Iterations copies of X
X ← Block
for i ← 0 to Iterations−1 do
   Vi ← X
   X ← BlockMix(X)

for i ← 0 to Iterations−1 do
   j ← Integerify(X) mod Iterations 
   X ← BlockMix(X xor Vj)

return X

Function BlockMix(B):

   The block B is r 128-byte chunks (which is equivalent of 2r 64-byte chunks)
   r ← Length(B) / 128;

   Treat B as an array of 2r 64-byte chunks
   [B0...B2r-1] ← B

   X ← B2r−1
   for i ← 0 to 2r−1 do
      X ← Salsa20/8(X xor Bi)  // Salsa20/8 hashes from 64-bytes to 64-bytes
      Yi ← X

return ← Y0∥Y2∥...∥Y2r−2 ∥ Y1∥Y3∥...∥Y2r−1
```

Scrypt被用在很多新的POW的虚拟货币中，用以表示他们挖矿程序的公平性，比如Tenebrix、 Litecoin 和 Dogecoin等。据说以太坊的 PoW 共识算法也是利用 Scrypt 实现的，但事实上以太坊自己实现了一套哈希算法，叫做 Ethash 。

> Scrypt 是一种资源消耗型的算法，但可以灵活地设定使用的内存大小

### 5. Argon2

2013 年，NIST（美国国家标准与技术研究院）举办了密码散列竞赛，宣布将选择一种新的标准算法，2015 年 Argon2 被宣布为最终获胜者，其他四种算法获得了特别认可：Catena、Lyra2、Makwa 和 yescrypt。

Argon2 的设计目标是实现最高的内存填充率和多个计算单元的有效使用，同时仍然提供对（GPU）权衡攻击的防御。
Argon2 基于 AES 实现，现代的 x64 和 ARM 处理器已经在指令集扩展中实现了它，从而大大缩小了普通系统和攻击者系统之间的性能差距。

Argon2 具有三个变体：Argon2i、Argon2d 和 Argon2id。

- Argon2d：速度最快，并且使用依赖于数据的内存访问，是对抗侧信道攻击的最安全选择，适用于加密货币等场景
- Argon2i：使用与数据无关的内存访问，速度最慢，内存消耗高，是抵抗 GPU 破解攻击的最安全选择，适用于密钥派生等场景
- Argon2id：是 Argon2i 和 Argon2d 的混合体，使用了数据依赖和数据无关的内存访问的组合

下图是最简单的，非并行的Argon2的算法流程。
![Argon2](argon2.png)
以下是使用方法的示例：

``` js
Inputs:
   password (P):       Bytes (0..232-1)    需要加密的原始信息P
   salt (S):           Bytes (8..232-1)    Salt盐，建议16个字节
   parallelism (p):    Number (1..224-1)   并行程度p，表示同时可以有多少独立的计算链同时运行
   tagLength (T):      Number (4..232-1)   指定返回的Tag密文长度
   memorySizeKB (m):   Number (8p..232-1)  内存大小, 单位是MB
   iterations (t):     Number (1..232-1)   迭代器的个数t，用于提升运行速度
   version (v):        Number (0x13)       版本号v，一个字节，取值0x13
   key (K):            Bytes (0..232-1)    可选的，安全值key (Errata: PDF says 0..32 bytes, RFC says 0..232 bytes)
   associatedData (X): Bytes (0..232-1)    可选的，附件数据
   hashType (y):       Number (0=Argon2d, 1=Argon2i, 2=Argon2id)  Argon2的类型
Output:
   tag:                Bytes (tagLength)   运算结算密文, 长度是tagLength
```

## 四、典型案例：Dropbox 的三层加密策略

Dropbox 公司曾公开分享过自己对用户账号的密码加密的策略，使用了三层加密，从里到外就像洋葱一样层层叠加。
![Dropbox](dropbox.png)

1. 首先，对明文密码使用 SHA512 散列算法，得到固定长度的 512 字节散列值。
   某些 Bcrypt 实现可能默认散列值长度为 72 字节，从而降低了密码的熵值；而有的则允许变长密码，容易受到 DoS 攻击。
2. 然后，针对散列值进行 Bcrypt 再次散列，每个密码都有不同的“盐”，并且是分开存储的
   Bcrypt 速度比较慢，这样就很难通过硬件加速来加快破解速度。
   Bcrypt 散列使用了成本因子 10（每个因子相当于每一步计算需要耗费 100 毫秒的时间），这样就更是加大了暴力破解的难度。
3. 最后，散列值会再次经过 AES256 算法的加密，这次加密会使用到对称秘钥，也就是所谓的“胡椒粉”（pepper）。
   pepper 与账号数据库分开存储，但所有账号的pepper都是统一的

---

## 参考文献

- [密码加密存储技术详解](https://www.ujcms.com/knowledge/509.html)
- [加盐密码哈希：如何正确使用](https://www.tomczhen.com/2016/10/10/hashing-security/)
- [PBKDF2算法原理](https://blog.csdn.net/HORHEART/article/details/119968850)
- [Salted Password Hashing - Doing it Right](https://crackstation.net/hashing-security.htm)
- [密码加密存储技术详解](https://www.ujcms.com/knowledge/509.html)
- [密码学系列之:加密货币中的scrypt算法](https://cloud.tencent.com/developer/article/1888720)
- [Colin Percival 关于 Scrypt 的原始论文](http://www.tarsnap.com/scrypt/scrypt.pdf)
- [Scrypt - RFC 7914 官方文档](https://www.rfc-editor.org/rfc/rfc7914)
- [Argon2 的技术白皮书 - 英文](https://github.com/P-H-C/phc-winner-argon2/blob/master/argon2-specs.pdf)
- [Argon2 的参考 C 实现 - Github](https://github.com/p-h-c/phc-winner-argon2)
- [什么是彩虹表 Rainbow Table](https://www.11meigui.com/2020/rainbow-table.html)
