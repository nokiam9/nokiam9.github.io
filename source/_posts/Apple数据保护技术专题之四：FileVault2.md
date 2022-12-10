---
title: Apple数据保护技术专题之四：FileVault2
date: 2022-12-08 11:19:15
tags:
---

从本质上说，FileVault 属于 FDE（Full Disk Encryption，全盘加密）技术，早期的 Android 设备也是采用此方案，但随着技术发展，Andriod 和 iOS 逐渐都演进为 FBE（File Based Encryption，文件加密），仅在 MacOS 中保留了 FileVault2，但同时也支持 DataProtection 数据保护技术。

## 一、Legacy FileVault

2003年，Mac OS X Panther (黑豹，10.3) 首次发布了 FileVault，当时的版本只加密了用户目录（而非启动宗卷），方法是采用一个巨大的稀疏磁盘镜像文件（sparse disk image）作为虚拟磁盘，负责存储加密的文件元数据。在启用 FileVault 功能时，系统会要求用户设置一个**主密码**，当用户忘记密码时，需要这个主密码或是恢复密钥直接解密文件。
> 与磁盘镜像文件`.dmg`不同，稀疏磁盘镜像文件`.sparseimage`的实际磁盘占用空间是根据使用情况动态扩展的

2006年，Mac OS X Leopard（花豹，10.5）做了少许改进，将虚拟磁盘改为稀疏绑定磁盘镜像（sparse bundle disk image），特点是将文件分割为若干个 8MB 的小文件（Band），在文件修改时无需复制全部内容，仅复制本次修改的 Band，因此备份速度更快，非常适合 Time Machine 的需求。

Apple 将上述两种版本的技术统称为legacy FileVault，其存在很多严重问题，包括：

- 只加密了用户目录的元数据，实际上数据文件的内容并未加密
- 采用 1024-bit RSA 或 3DES-EDE 算法已被证明存在破解风险，加密算法 CBC 模式的安全强度不足
- 当 MacOS 置于睡眠模式时，密钥存在的泄露风险

### 二、FileVault 2

2010年，Mac OS X Lion (狮子，10.7) 发布了重新设计的 FDE 方案 - FileVault 2，其技术特点包括：

1. 启动卷宗的分区由 HFS+ 文件系统改为一个 CoreStorage 管理的加密卷，并增加了一个 Recovery HD 分区卷
2. 使用**用户登录密码**作为加密令牌，
3. 加密标准采用 NIST 推荐的 AES-128-AES 模式，分为并采用AES的XTS-AES模式将数据划分成128位的块，同时生成一枚256位密钥来加密磁盘，这一标准也是NIST推荐的标准。当用户解锁后，其他用户也可以访问这些数据，直到电脑关机。

> CBC 引入了一个IV值递归　上一组加密后产生新的IV值作为下一数据块加密的输入 这样至少有个一个参数不同　但带来的问题是没有办法随机访问各个区块了，访问某个区块必须解出前一个区块数据。
> XTS 模式加入了 tweak key 与 AES key 互相配合，各个区块用同样的AES key 但Tweak key 不相同 比如设置为与区块的index 成对应关系, 这样各个区块数据加密解密互相独立（独立的tweak key)
> 
抛弃了基于 HFS+ 文件系统，通过磁盘镜像对 HFS+ 文件系统的元数据加密的技术路线，采用了全新设计的




- 直接加密整个 Mac OS 的启动宗卷（自然也包括用户目录），并
- 采用这种加密方法，已授权的用户信息将从一个单独的非加密引导卷宗读取。(分区格式为 Apple_Boot)。

FileVault

约有3%的I/O的性能下降会出现在使用AES指令集的CPU上（如Intel Broadwell 架构CPU， OS X 10.10.3环境），早期的酷睿CPU，或者其他不使用该指令集的处理器会有更明显的性能下降。

系统运行时执行 FileVault2，电脑会创建一个**恢复密钥**，并在屏幕上显示出来，提示用户保管，并且提供了一个将密钥上传至 Apple 的可选项。恢复密钥共有120位，由全部英文字母和数字 1-9 组成，系统会调用/dev/random的随机数生成器生成整枚密钥。因此其安全性取决于macOS 使用的随机数生成算法，根据密码学家 2012 年的声明指出，该算法是安全的。

除非重新加密整个卷宗，否则将无法修改恢复密钥。

![Volume Main Key](vmk.png)

在了解了 FileVault2 加密的流程之后，可以根据具体方 法得到相对应的解密流程。将加密后的磁盘用外部工具加 载观察，从磁盘结构来看，加密后的硬盘与加密前硬盘发 生了重大的变化。通过对加密数据研究和相关资料文献的阅读，得 知加密卷的卷头存储了解密所需的各种相关参数 [4]。图 2 为 FileVault2 解密流程。

根据对加密卷数据分析可得加密卷的头部主要包含有 如下信息 :
卷头部签名、块大小、卷大小、元数据大小、第一个 元数据块块号、第二个元数据块块号、第三个元数据块块号、 第四个元数据块块号、加密方法、逻辑卷 ID(用于解密加 密的密钥文件)、物理卷 ID[5]。
解密步骤如下 :
1)识别到 FileVault2 加密卷和存储的需要提取的用户 密码生成的主密钥信息或者恢复令牌的 EncryptedRoot.plist. wipekey 文件 ;
2)解密通过文件系统解析获得的 EncryptedRoot.plist. wipekey 文件后，提取存储的密钥信息 ;
3)输入用户登录密码或者恢复密钥，验证步骤 2)中 提取到的密钥信息是否正确 ;
4)验证正确后根据用户密码或者恢复密钥及加密的 EncryptedRoot.plist.wipekey 文件中存储的密钥信息生成主 密钥和 tweak key ;
5)用生成的两个解密密钥信息(key1 和 key2)及加密 卷数据进行解密，生成解密数据即完成解密。
用取证工具或其他可以解析 HFSPlus 文件系统的工具 对解密数据进行解析，可以验证解密结果是否正确。


![密钥层次架构](arch.png)


### VEK - Volume Encryption Key，卷宗封装密钥

---

关键文件：EncryptedRoot.plist

Once decrypted, the file EncryptedRoot.plist has an XML structure with the following important entries:
• PassphraseWrappedKEKStruct(1)
– 284 byte structure for recovery password.
• PassphraseWrappedKEKStruct(2)
– 284 byte structure for user password.
• KEKWrappedVolumeKeyStruct(1) – unused.
• KEKWrappedVolumeKeyStruct(2)
– contains wrapped volume master key.


``` c
p = get_user_password()                                 // 用户输入password 
salt = get_salt_from_PassphraseWrappedKEK()
iterations = 41000
pk = pbkdf2(p, salt, iterations, HMAC-SHA256)           // 计算PDK
kek_wrapped = get_kek_from_PassphraseWrappedKEK()           
kek = aes_unwrap(kek_wrapped, pk)                       // 获得
vmk_wrapped = get_vmk_from_KEKWrappedVolumeKey()
vmk = aes_unwrap(vmk_wrapped, kek)
```

## Android 的 FDE 历程

FDE加密又是怎么一回事呢，看看google怎么定义的吧（以下部分内容引用网络某大神的解释）：

- 安卓会通过一个随机生成的128位设备加密密钥 (Device Encryption Key, DEK) 来加密设备的文件系统。
- 安卓使用用户的PIN或者密码来加密DEK，并将它存储在设备加密过的文件系统上。从物理上来讲，它也在设备的闪存芯片中。当你输入正确的PIN或密码时，设备可以解锁DEK，并使用密钥来解锁文件系统。

- 实际上，DEK是使用用户的PIN或密码，外加一个被称为KeyMaster Key Blob的加密数据块来进行加密的。这个数据块包含一个由KeyMaster程序生成的2048位RSA密钥，它运行在设备处理器上的一个安全区域上。KeyMaster会创建RSA密钥，将其存储在数据块中，并为安卓系统创建一份加密过的拷贝版本。
    安卓系统和你的移动应用运行在处理器的非安全区域上。安卓没有访问KeyMaster的安全世界的权限，因此它无法知晓数据块里的RSA密钥。安卓只能获得这个数据块的加密版本，而只有KeyMaster能够解密它。

- 当你输入PIN或密码时，安卓拿到加密过的数据块，并将它和使用scrypt处理过的PIN或密码一起，传回运行在处理器安全区域上的KeyMaster。KeyMaster将私密地使用处理器中带有的私钥来对数据块进行解密，获得长RSA密钥。然后，它将私密地使用scrypt处理过的PIN或密码，外加长RSA密钥，来制造一个RSA签名，并将签名发回给安卓。之后安卓使用一系列算法来处理这一签名，并最终解密DEK，解锁设备。

因此，全部流程都基于KeyMaster的数据块。数据块包含解密DEK所需的长RSA密钥。安卓只拥有加密后的数据块，而只有用户才有PIN或密码。此外，只有KeyMaster才能解密加密过的数据块。

### Google 的版本历史

- Android 4：首次引入 FDE 的 beta版本，由于 CPU 普遍不支持硬件加密严重影响速度，并未普遍应用
- Android 5: Google 推出 5.0 强制采用 FDE 加密，由于终端厂商的普遍抵制又推出 5.1 版本，改为用户自行设置是否启用
- Android 6: 高通发布骁龙810支持硬件加密，6.0 版本终于强制采用 FDE 加密

### 5.0中的加密 磁盘加密密钥 的实现逻辑

1. 产生随机16 Bytes DEK(disk encryption key--磁盘加密用的密钥)及16 Bytes SALT；
2. 对(用户密码+SALT)使用scrypt算法产生32 Bytes HASH 作为IK1(intermediate key 1);
3. 将IK1填充到硬件产生的私钥规格大小(目前看到是RSA算法，256Bytes), 具体是: 
00 || IK1 || 00..00  ## one zero byte, 32 IK1 bytes, 223 zero bytes.
4. 使用硬件私钥 HBK 对 IK1 进行签名，生成256 Bytes签名数据作为IK2；
5. 对(IK2+SALT)使用scrypt算法(与第二步中的SALT相同)产生出32 Bytes HASH 作为IK3；
6. 使用IK3前16 Bytes作为KEK(用来加密主密钥DEK的KEY)，后16 Bytes作为算法IV(初始化向量)；
7. 使用AES_CBC算法，采用KEK作为密钥，IV作为初始化向量来加密用户的主密钥DEK，生成加密后的主密钥，存入分区尾部数据结构中；

---

Secure Startup
Secure Startup 為 FDE 的延伸版本，唯一的差別是保護加密/解密的 Masterkey 金鑰的方式不同。原本在 FDE 架構下保護 Masterkey 的其中一個密碼為 default_password，在 Secure Startup 下可改為用戶自訂的螢幕鎖密碼。


## AFPS

![AFPS](apfs_concepts.png)

[https://bombich.com/kb/ccc5/working-apfs-volume-groups](https://bombich.com/kb/ccc5/working-apfs-volume-groups)
[https://www.ntfs.com/apfs-structure.htm](https://www.ntfs.com/apfs-structure.htm)
---

## 附录一：AES 的

AES五种加密模式（CBC、ECB、CTR、OCF、CFB）.
参考文档: [https://csrc.nist.gov/publications/detail/sp/800-38a/final](https://csrc.nist.gov/publications/detail/sp/800-38a/final)

[分组密码工作模式 - WiKi](https://zh.m.wikipedia.org/zh-hans/%E5%88%86%E7%BB%84%E5%AF%86%E7%A0%81%E5%B7%A5%E4%BD%9C%E6%A8%A1%E5%BC%8F)

### 1. ECB（Electronic Codebook Book，电码本模式)

这是最简单的模式，将整个明文分成若干段相同的小段，然后使用同一个密钥对每一小段进行加密，加密和解密都可以并行计算。
在 ECB 模式下，相同的明文块会被加密成相同的密文块，强烈建议**不要使用 ECB 模式！它可能会危及整个加密方案**。

### 2. CTR (Counter，分组模式）

同样将明文/密文拆分成若干个长度固定的分组，但是其算法包含了3个参数：

- 初始向量 IV（Initialization Vector）：也被称作 Salt 或者 Nonce。通常是一个随机数，主要作用是往密文中添加随机性，使同样的明文被多次加密也会产生不同的密文，从而确保密文的不可预测性
- 计数器 Counter: 自增的算子。一般从 0 开始，每次计算都自增 1，相当于一次一密
- 密钥 Key: 对称加密的密钥

CTR模式简单快速，安全可靠，而且可以并行加密，但是在计算器不能维持很长的情况下，密钥只能使用一次。

### 3. CBC（Cipher Block Chaining，密码分组链接模式）

在CBC模式中，。同时，为了保证每条消息的唯一性，在第一个块中需要使用初始化向量。

同样将明文/密文拆分成若干个长度固定的分组，其算法规定：

- 第一个分组需要使用初始化向量 IV 执行异或操作
- 随后的每个明文块，先与前一个密文块进行异或后，再进行加密。即每个密文块都依赖于它前面的所有明文块、

CBC 模式有较好的保密性，但由于依赖关系无法实现并行计算，因此不适合对流数据进行加密。

CBC 模式有较好的保密性，是最常见的工作模式。主要缺点在于加密过程存在依赖关系，无法被并行化，但是解密过程可以被并行化！！！

### 2. GCM (Galois/Counter，分组模式）

GCM (Galois/Counter) 模式在 CTR 模式的基础上，添加了消息认证的功能，而且同时还具有与 CTR 模式相同的并行计算能力。因此相比 CTR 模式，GCM 不仅速度一样快，还能额外提供对消息完整性、真实性的验证能力。





### 5. CFB（Cipher FeedBack，密码反馈模式）

类似于CBC，可以将块密码变为自同步的流密码；工作过程亦非常相似，CFB的解密过程几乎就是颠倒的CBC的加密过程：

### 6. OFB（Output FeedBack，输出反馈模式）

这种模式较复杂。

常用的安全块模式是 CBC（密码块链接）、CTR（计数器）和 GCM（伽罗瓦/计数器模式），它们需要一个随机（不可预测的）初始化向量 (IV)，也称为 nonce 或 salt
「CTR（Counter）」块模式在大多数情况下是一个不错的选择，因为它具有很强的安全性和并行处理能力，允许任意输入数据长度（无填充）。但它不提供身份验证和完整性，只提供加密

GCM（Galois/Counter Mode）块模式继承了 CTR 模式的所有优点，并增加了加密消息认证能力。GCM 是在对称密码中实现认证加密的快速有效的方法，强烈推荐

CBC 模式在固定大小的分组上工作。因此，在将输入数据拆分为分组后，应使用填充算法使最后一个分组的长度一致。大多数应用程序使用 PKCS7 填充方案或 ANSI X.923. 在某些情况下，CBC 阻塞模式可能容易受到「padding oracle」攻击，因此最好避免使用 CBC 模式
众所周知的不安全块模式是 ECB（电子密码本），它将相等的输入块加密为相等的输出块（无加密扩散能力）。
CBC、CTR 和 GCM 模式等大多数块都支持「随机访问」解密。比如在视频播放器中的任意时间偏移处寻找，播放加密的视频流
总之，建议使用 CTR (Counter) 或 GCM (Galois/Counter) 分组模式。 其他的分组在某些情况下可能会有所帮助，但很可能有安全隐患，因此除非你很清楚自己在做什么，否则不要使用其他分组模式！

CTR 和 GCM 加密模式有很多优点：它们是安全的（目前没有已知的重大缺陷），可以加密任意长度的数据而无需填充，可以并行加密和解密分组（在多核 CPU 中）并可以直接解密任意一个密文分组。 因此它们适用于加密加密钱包、文档和流视频（用户可以按时间查找）。 GCM 还提供消息认证，是一般情况下密码块模式的推荐选择。

请注意，GCM、CTR 和其他分组模式会泄漏原始消息的长度，因为它们生成的密文长度与明文消息的长度相同。 如果您想避免泄露原始明文长度，可以在加密前向明文添加一些随机字节（额外的填充数据），并在解密后将其删除。


分组密码有五种工作体制：1.电码本模式（Electronic Codebook Book (ECB)）；2.密码分组链接模式（Cipher Block Chaining (CBC)）；3.计算器模式（Counter (CTR)）；4.密码反馈模式（Cipher FeedBack (CFB)）；5.输出反馈模式（Output FeedBack (OFB)）。
以下逐一介绍一下：
1.电码本模式（Electronic Codebook Book (ECB)
    这种模式是将整个明文分成若干段相同的小段，然后对每一小段进行加密。

AES 是对称加密算法的一种，了解AES 算法可以从这几方面入手：

密钥支持的长度
常用的工作模式（ECB模式/CBC模式）
Padding 的填充模式
密钥
密钥是AES算法实现加解密的根本。对称加密算法之所以对称，是因为这类算法对明文的加密和解密需要使用相同的一个。
AES 支持三种长度的密钥:

128bit (16B), 192bit(24B) , 256bit(32B)

使用效果:

AES256 安全性最高，AES128性能最优。本质是它们的加密处理轮数不同。

AES128	10轮
AES192	12轮
AES256	14轮
ECB 模式
ECB 模式是最简单块密码加密模式，加密前根据加密块大小（AES 128位）分成若干块，之后将每块使用相同的密钥单独加密，在该模式下，每个明文块的加密都是独立的，互不影响的。解密同理。

优势

简单

有利于并行计算

缺点

相同的明文块经过加密会变成相同的密文块，因此安全性较差。
CBC 模式
CBC模式引入一个新的概念：初始向量IV。

IV的作用和MD5的"加盐"有些类似，目的是防止同样的明文块始终加密成相同的密文块。

CBC模式原理:

在每个明文块加密前会让那个明文块和IV向量先做异或操作。IV作为初始化变量，参与第一个明文块的异或，

后续的每个明文块和它前一个明文块所加密出的密文块相异或，这样相同的明文块加密出来的密文块显然不一样。

优势

安全性更高
缺点

无法并行计算，性能上不如ECB

引入初始向量IV，增加复杂度

#### CTS

在密码学中，密文窃取（CTS）是使用分组密码操作模式的通用方法，该操作模式允许处理不能均匀分割成块的消息，而不会延长密文，代价是略为复杂。

目录
一般特性
编辑
窃取密文是使用密码加密明文的技术，不须将消息填充到块大小的倍数，因此密文与明文大小相同。

它通过更改消息最后两块来实现这点。除了最后两块之外，所有块都保持不变，但“窃取”倒数第二块的一部分密文用以填充最后一块明文块。填充的最后一块，然后像往常一样加密。

最终密文的最后两块，包括部分倒数第二块（删掉“窃取”部分）和完整的最后一块，它们大小与原明文相同。

解密时要求首先解密最后一块，然后将“窃取”的密文恢复到倒数第二块，然后可以像往常一样解密。

原则上，任何使用块密码的分组加密模式都可用，但流密码模式已经可以加密任意长度的消息无需填充，因此它们不能用该操作。与窃取密文相结合的常用加密方式有电子密码本（ECB）和密码块链接（CBC）。

ECB密文窃取要明文长过一块。当明文长度为一个或更少时，一种可能的解决办法是，使用一种类似流密码的分组密码操作模式，如CTR、CFB或OFB。

CBC密文窃取不一定要明文长过一块。在明文为一块或更少块长度的情况下，初始向量（IV）可作为先前的密文块。在这种情况，必须将修改后的IV发送予接受者。但这在发送密文时IV不能由发送者自由选择的情况下（如当IV是派生值或预先确定的值）不太可能，并且在这种情况下，针对CBC模式的密文窃取只能在明长于一个块文中发生。

为了以CTS加密或解密未知长度的数据，必须延迟处理（和缓存）最新的两块数据块，以便处理数据流末端。

## 参考文献

Stephen Foskett 是一名存储技术专家，其个人博客对 FileVault 进行了非常有价值的分析。
请参见[MacOS X 的 Corestorage 逻辑卷组管理器](https://blog.fosketts.net/2011/08/04/mac-osx-lion-corestorage-volume-manager/)

- [MacOS 的安全和隐私指南](https://github.com/drduh/macOS-Security-and-Privacy-Guide/blob/master/README-cn.md)
- [FileVault2加密分区离线解密技术及其取证应用 - 蓝朝祥](FileVault2加密分区离线解密技术及其取证应用_蓝朝祥.pdf)
- [Security Analysis and Decryption of FileVault 2 - Omar Choudary](slides_fv2_ifip_2013.pdf)
- [Infiltrate the Vault: Security Analysis and Decryption of Lion Full Disk Encryption - Omar Choudary](2012-374.pdf)
- [你的安卓手机究竟是FDE加密还是FBE加密？](https://page.om.qq.com/page/O3yauEIx2l-9WrUkHgQgRUBw0)
- [android FDE功能介绍](https://blog.csdn.net/bob_fly1984/article/details/80369900)
