---
title: 流密码算法的硬件实现-以GSM网络的A5算法为例
date: 2022-12-23 17:17:12
tags:
---
1917 年, Vernam 密码体制被提出, 后来 Mauborgne 提出了改进方案, 即 “一次一密” 密码体制。
1949 年, Shannon 证明 “一次一密” 密码体制具备完全的理论安全性，但必须满足三个条件：

- 密钥流必须完全随机生成
- 密钥长度至少与明文相同
- 密钥不能重复使用

这些限制条件在实际应用中存在很大困难，而且密钥的生成、分配和管理也是不容忽视的问题，因此科学家设计了各种流密码（Stream cipher，也称为流加密）算法来代替 “一次一密” 体制，其基本特征是：

- 作为一种对称加密算法，加密和解密双方使用相同的伪随机加密数据流（pseudo-random stream）作为密钥流（Keystream）
- 明文数据（Plaintext）按 bit 位与密钥流**顺次**对应执行异或（XOR）操作，得到密文数据流（Cipertext）

为了提高处理性能，伪随机密钥流（keystream）通常由一个随机的种子（seed）通过 PRG 算法（pseudo-random generator）生成，即由较短的数据流通过特定算法得到较长的密钥流，因此 PRG 算法的不可预测性成为确保流加密安全性的关键问题。

## 一、GSM 安全体系

GSM 的安全性基于对称密钥的加密体系，定义了三种加密算法：

- A3：用于移动设备到 GSM 网络认证的算法，负责生成认证码 SRES
- A5：用于认证成功后加密语音和数据的算法，负责生成会话密钥（一次一密）
- A8：用于产生对称密钥的密钥生成算法，负责生成通信密钥 Kc

SIM 是 GSM 终端的关键安全载体，负责实现 A3 和 A8 算法，并负责持久化存储核心数据：

- IMSI：SIM 卡的全球唯一的标志号。手机在开机时，一次性从卡里面读出并发给移动网络，鉴权成功后生成并对外提供 TMSI
- Ki：SIM 卡的唯一根密钥，16字节长度，长期存储。无法通过 SIM 卡的接口读出，只能用于生成密钥，例如 Kc

GSM 的各个网元分别提供相关安全能力：

- 移动终端：负责实现 A5 算法，并持有 TMSI。
- HLR/AUC： 保存 IMSI 和手机号的对应关系、IMSI 和 Ki 的对应关系，提供 RAND 随机数，负责实现 A3 和 A8 算法
- MSC：持有 IMSI 并生成 TMSI，管理认证向量（RAND、SRES、Ki），也称为**认证三元组**
- BST：负责实现 A5 算法
- EIR：设备标识寄存器，负责核对 IMEI

![A3 & A8](COMP128.png)

在手机登录移动网络的时候，移动网络会产生一个 16 字节的随机数据 RAND 发给手机，手机将这个数据发给 SIM 卡， SIM 卡用自己的密钥 Ki 和 RAND 做运算以后，生成一个 4 字节的应答 SRES 发回给手机，并转发给移动网络，与此同时，移动网络也进行了相同算法的运算，移动网络会比较一下这两个结果是否相同，相同就表明这个卡是我发出来的，允许其登录。这个就是 GSM 规范的 A3 算法，m = 128 bit, k = 128 bit, c = 32 bit，很显然，这个算法要求已知 m 和 k 可以很简单的算出 c ，但是已知 m 和 c 却很难算出 k 。A3 算法是做在 SIM 卡里面的，因此如果运营商想更换加密算法，他只要发行自己的 SIM 卡，让自己的基站和 SIM 卡都使用相同的算法就可以了，手机完全不用换。

在移动网络发送 RAND 过来的时候，手机还会让 SIM 卡对 RAND 和 Ki 计算出另一个密钥以供全程通信加密使用，这个密钥的长度是 64 bits, 通常叫做 **Kc**, 生成 Kc 的算法是 A8 算法 ，因为 A3 算法和 A8 算法接受的输入完全相同，所以实现者偷了个懒，用一个算法 COMP128 同时生成 SRES 和 Kc 。

## 二、A5 算法原理

由于移动终端和网络基站之间的无线数据传输完全暴露在开放环境，空中接口的安全问题必须得到解决。第一代移动通信实现了控制信道的数字化，但业务信道由于仍然采用模拟技术因而无法实现加密，GSM 系统通过业务信道的数字化并采用了流密码技术，有效解决了传输安全问题，因此迅速得到普及推广，其中 A5 算法功不可没。
根据 GSM 规范，先后发布了7个版本的 A5 算法，但 A5/1 是第一个版本，得到了最广泛的应用，如果没有特别说明，通常所说的 A5 就是指 A5/1。

加解密处理需要消耗大量的计算资源，早期的流密码大多基于专用硬件实现，最典型的就是 LFSR - 线性反馈移位寄存器，A5/1 算法就是如此。
1987年，法国开发了 A5/1 算法，用于对从电话到基站连接的加密处理，效率很高并符合统计检验要求。起初该算法是保密的，但最终不慎泄漏。

![A5](a5-arch.png)

### 1. LFSR - 线性反馈移位寄存器

A5/1 算法使用3个线性反馈移位寄存器，位长分别为 19 + 22 + 23 = 64。这个设计绝非偶然，正好用以容纳 64位 的初始密钥 Kc 。

|寄存器编号|寄存器位数|钟控位|抽头位|
|:-:|:-:|:-:|:-:|
|1|19|8|13, 16, 17, 18|
|2|22|10|20, 21|
|3|23|10|7, 20, 21, 22|

那么，这三个LFSR是如何生成密钥流的呢？对于每一个寄存器，它会通过向最高位的移动(产下处于最高位的孩子)来产生密钥，3个寄存器都通过这种操作来产生密钥，然后3个最高位经过异或操作得到最终的密钥。
下一个问题是如何补充最低位呢？答案是不同的寄存器采取不同的抽头策略，1号寄存器是计算第18、17、16、13位（**图中的蓝色位**）的异或值作为下一状态的最低位，2号寄存器是第21、20位的异或值，3号寄存器是第22、21、20、7位的异或值。

### 2. 钟控方式

为了提高复杂性以弱化线性漏洞，A5算法采用了一种基于**择多原则**的钟控方式，即在会话密钥的产生过程中，并非每个 LFSR 每次都会移动。

- 每个寄存器都有一个相关的钟控位，分别是第8、10、10位（**图中的黄色位**）
- 在每个周期，检查三个寄存器的钟控位，并确定多数位（0或者1）
- 对于每个寄存器，如果钟控位与多数位一致，则对寄存器作移位操作；否则，寄存器不移动，也就是前后两次提供的密钥是相同的

由于寄存器的个数是奇数，因此择多原则必然生效，不会存在无法判决的情况；同时，每次必然有2-3个寄存器发生移位，并且每个寄存器移位的概率都是3/4。
描述为：给定三个二进制位 x 、 y 和 z ，定义多数投票函数 maj(x, y, z)。也就是说，如果 x 、 y 和 z 的多数为 0，那么函数返回 0；否则，函数返回 1。

### 3. 工作流程

根据 GSM 系统的定义，每 20ms 的模拟话音进行一次采样，通过信源编码处理成 260bit 的数字信号；再经过信道编码变换为 456bit 的数字信号，然后经过交织分成8个部分，每部分 57bit ；空中的时隙中，每个时隙承载2部分即 114个bit 的内容，于是 20ms 的话音完全通过4个时隙的资源承载。因此，明文是一个 114位 的比特流，需要相应提供一个 114位 的密钥流。

1. 密钥导入：三个 LFSR 清零复位，执行`XOR + Shift`动作 64 次以导入密钥；
2. 随机数引入：以帧序号作为输入，三个 LFSR 执行`XOR + Shift`动作 22 次以导入随机数；
3. 概率校正：三个 LFSR 执行钟控动作 100 次，但不输出乱数；
4. 上行密钥流生成：三个 LFSR 执行钟控动作 114 次，每次动作后将三个 LFSR 的最高位输出并执行 XOR 作为上行密钥流输出；
5. 下行密钥流生成：重复步骤4，并将输出作为下行密钥流。

> TDMA 帧号以 2715648 为周期循环编号（22位），2<sup>21</sup>=2097152 < 2715648 > 4194304=2<sup>22</sup>
> 构造方式 `2715648 = 2048 * 26 * 51` ：基帧（8个时隙，4.615ms）—> 复帧（业务为26帧，控制为51帧）—> 超帧（26*51个帧）—> 巨帧（2048个超帧）

### 4. 破解之路

A5/1 算法曾经是使用最广泛的GSM加密算法，在设备中的支持程度也是几种算法中最高的。该算法在1994年被初步泄露，在1999年通过逆向工程的方式被公布，2000年开始被逐步破解。

- 2000年，Alex Biryukov 等通过构造庞大的具备初步彩虹表概念的数据库，通过查表取代计算的方式，以空间换时间，对A5/1实施已知明文攻击，但该方法需要首先通过大约248次运算，处理约300GB的数据
- 2007年，德国 Bochum 大学搭建了具有120个 FPGA 节点的阵列加速器，对包括A5/1算法在内的多种算法进行破解，由于采用FPGA成本较低，使A5/1算法破解在商业上成为可能
- 2009年，Karsten Nohl 等人利用3个月时间制作了2TB的彩虹表，并宣布利用P2P分布式网络下的Nvidia GPU显卡阵列即可破解A5/1算法
- 2016年，新加坡科技研究局用55天创建了一个 984GB 的彩虹表，通过使用3块 Nvidia GPU 显卡构成的计算装置在9秒内完成对A5/1算法的破解
- 此外，根据斯诺登披露的文件显示，美国NSA是可以破解A5/1算法的。

1993年，由于受巴统限制，A5/1 算法作为出口管制技术无法集成到在中国境内使用的设备中，为此开发了弱化版本的 A5/2 算法。该密码基于 4个 LFSR 的组合，具有不规则时钟和一个非线性组合器。
1999年，该算法被发现存在缺陷（后门）可以被实时破解，随后 3GPP标准 TS 33.020 中明确禁止使用。

## 三、移动通信网的密码算法演进

|加密算法|私有|私有|KAUSMI|KAUSMI|SNOW 3G|AES|ZUC|
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|算法类型|流密码|流密码|分组密码|分组密码|流密码|分组密码|流密码|
|密钥长度|64|64|64|128|128|128|128|
|工作模式|XOR|XOR|f8-mode|f8-mode|XOR|CTR|XOR|
|GSM(2G)|A5/1|A5/2|A5/3|A5/4
|GPRS (2.5G)|GEA1|GEA2|GEA3|GEA4|
|UMTS(3G)||||UEA1|UEA2|
|LTE(4G)|||||128-EEA1|128-EEA2|128-EEA3|
|NR(5G)|||||128-NEA1|128-NEA2|128-NEA3|

> GSM是一个面向语音通信的网络，2.5G 增加了分组交换的核心网 GPRS，采用的加密算法仍然是 A5 系列算法，但名称改为 GEA（GPRS Encryption Algorithm）。

### 1. KASUMI 算法

不同于流密码性质的 A5/1 和 A5/2 算法，GSM 规范还纳入了 A5/3 和 A5/4 算法，他们都是基于 KASUMI 的分组密码。
KASUMI 是日本三菱的 Matsui 等人基于 MISTY 算法改进而来，采用 Feistel 结构，密钥长度为128比特，对一个64比特的输入分组进行八轮的迭代运算，产生长度为64比特的输出。轮函数包括一个输入输出为32比特的非线性混合函数 FO 和一个输入输出为32比特的线性混合函数 FL。函数 FO 由一个输入输出为16比特的非线性混合函数 FI 进行3轮重复运算而构成。而函数 Fl 是由使用非线性的S盒 S7 和 S9 构成的4轮结构。

A5/3 算法的分组大小 64bit，密钥长度 64位。为了实现流密码能力，其工作模式为 F8 机密性算法和 F9 完整性算法，其中 F8 是变形的 OFB 模式，F9 是变形的 CBC-MAC 模式。
2001年开始，A5/3 算法不断受到安全性挑战，为此将密钥长度扩展到了128位并推出 A5/4 算法。
2015年，A5/4 算法仍然遭到破解，为此 4G 标准制定时 KASUMI 被完全放弃。

### 2. SNOW 3G 算法

SNOW 1.0是欧洲的 NESSIE（New European Schemes for Signatures, Integrity and Encryption）项目中产生的候选算法，是一种 128位 的流加密算法。
SNOW 1.0算法在公开后，被发现存在一些算法缺陷，之后经过不断修订和增强，从 SNOW 1.0，SNOW 2.0直到 SNOW 3G被认可作为 3G 使用的第二种算法 UEA2。
后续，4G 标准中沿用为 128-EEA1，5G标准中沿用为 128-NEA2。

### 3. AES 算法

在 4G 标准中，AES-128 分组算法（ CTR 工作模式）被引入并命名为 128-EEA2，后续 5G 标准继续沿用为 128-NEA2。
采用 AES 取代 KASUMI 主要有以下原因：

- 4G的基站需要实现 NDS/IP 的保护，而 NDS/IP 中需要使用 AES，所以4G的基站天然支持AES算法；
- KASUMI 算法有授权费用，虽然使用KASUMI进行完整性保护不需要缴费，但若用于加密则需要缴费；
- 此外，4G 支持非 3GPP 接入，而 AES 在非 3GPP 接入（例如 WLAN）场景中的应用更广。

### 4. ZUC 算法

2009年，在3GPP SA3立项讨论时，考虑到中国自主加密算法的需求，在工信部和国家密码局的领导下，由信通院牵头，中国移动、中国联通、大唐、华为、中兴、中科院软件所等参与成立中国自主加密算法推进工作组。
2011年，中科院软件所设计的祖冲之算法（代号ZUC）正式纳入3GPP标准，在LTE Rel-11版本中成为可选的第3种加密算法。祖冲之算法为流加密算法，具备与SNOW 3G基本等同的性能和加密强度。

## 四、总结分析

从移动通信网络的加密技术演进中分析，传统的流密码技术正在被分组加密技术逐渐替代。

1. 随着硬件处理能力的高速发展，基于通用 CPU 的分组加密算法和基于专用硬件的流密码算法的性能差距几乎消失，而加密专用指令集进一步扩大了分组密码的技术优势和成本优势；
2. 随着网络数据流量的高速增长，伪随机密钥流的构造难度越来越高，流密码算法被攻击破解的风险显著增加，而种类丰富的分组密码算法可以提供更高强度的安全性；
3. 随着加密密钥长度的不断增加（64-128-256），许多流密码算法也从按 bit 顺序加密，转向按 byte 顺序加密，几乎等价于 8位 的分组加密方案，也说明了两者正在逐步融合；
4. OFB、CFB 和 CTR等工作模式技术的出现，可以很方便地将分组密码算法**转换**为流密码算法

---

## 参考文献

- [移动通信网中的密码算法演进之一：机密性保护](https://www.sohu.com/a/237953596_556637)
- [GSM加密系统里涉及的三种算法：A3算法、A5算法和A8算法](https://www.jiamisoft.com/blog/22123-gsma.html)
- [A5算法 - WiKi](https://zh.wikipedia.org/zh-sg/A5/1)
- [密码系列2-流密码实战之A5算法](http://kenshichong.github.io/2015/12/26/%E5%AF%86%E7%A0%81%E7%B3%BB%E5%88%972-%E6%B5%81%E5%AF%86%E7%A0%81%E5%AE%9E%E6%88%98%E4%B9%8BA5%E7%AE%97%E6%B3%95/)
- [ZUC祖冲之序列密码算法](https://www.cnblogs.com/mengsuenyan/p/13819504.html)
- [ZUC(祖冲之)算法](https://juejin.cn/post/7118582525424828424)
- [3G核心加密算法之KASUMI加密算法](https://www.jiamisoft.com/blog/2669-kasumijiamisuanfa.html)

### 文档下载

- [流密码算法、架构与硬件实现研究_赵石磊](流密码算法、架构与硬件实现研究_赵石磊.pdf)
- [GSM系统安全体系 - 2004英文版](Security_in_the_GSM_system_20040105.pdf)
- [A3/A8 & COMP128 技术破解](S5.Brumley-comp128.pdf)
- [A5/1 算法可抵抗相关攻击的改进方法 - 张伟](A5-1算法可抵抗相关攻击的改进方法-张伟.pdf)
- [a5/2 算法的破解分析](a52-slides.pdf)
- [Instant Ciphertext-Only Cryptanalysis of GSM Encrypted Communication](978-3-540-45146-4_35.pdf)
