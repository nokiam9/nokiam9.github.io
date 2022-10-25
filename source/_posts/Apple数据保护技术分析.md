---
title: Apple数据保护技术分析
date: 2022-10-08 20:05:45
tags:
---

花了好长时间，才明白Apple公司数据保护（Data Protection）的核心。

根据Apple安全白皮书的官方文档，数据保护实际上有两条路径，一是面向智能终端的 iOS 发展来的 Data Protection，二是面向个人电脑的 MacOS 发展来的数据保险箱，由于依赖Intel CPU，数据安全就是各种打补丁。M1 发布后两条路径开始融合，Secure Encalve 和 AFPS 成为融合的基石。
任何技术都有路径依赖，虽然 HFS+ 被认为是业界最烂的文件系统，但其许多重要的核心功能仍然需要 AFPS 继承下来，当然也为后续发展打下基础。

## 一、基础原理

![更细架构](arch2.png)

- 每个文件创建时会生成一个`Per-file Key`，在文件写入闪存时通过硬件用`AES-XTS`算法加密
- `Per-file Key`存储在文件元数据 Metadata 中，被`Class Key`和`Filesystem Key`加密
- `Class Key`和`Filesystem Key`都被硬件密钥`UID`保护
- `Class Key`还被用户密码`Passcode`保护

实际上，Apple就是在 HFS+ 文件系统的 inode 中增加了一个`cprotect` 字段：

- 如果数据保护级别是 A / B / C，存储以`Class Key`加密的`Per-file Key`
- 如果数据保护级别是 D，存储加密后的`DKey`(Device Key，正好也是D)
- 如果设置为`discarded`，则意味着密码失效，文件就永远无法打开，也是数据被抹去了...

## 一. 安全隔区访问的`Secure Nonvolatile Storage`是什么？

![基本结构](arch3.jpg)

CPU都是通用的，这些关键密钥都是个性化数据，必须有一个存储设备提供数据持久化！
秘密就藏在 NAND 闪存的1号扇区，Apple 将它作为隐藏的可安全擦除区域(Effaceable Storage)，就是安全隔区的技术架构中的`Secure nonvolatile storage`。
> Effaceable Storage：accesses the underlying storage technology (for example, NAND) to directly address and erase a small number of blocks at a very low level.

当然，Apple官方文档说安全隔区通过专用 I2C 总线连接，但 NAND 闪存不太可能再增加一个物理接口，也许是复用吧。
这个隐藏存储区域的容量是 960 Bytes，存储了3个关键数据。

### EMF：文件系统主密钥，File-system Master Encryption key（反序？）

- 功能描述：**数据分区**的加密密钥，也被称为媒体密钥（media key）；iOS 5之后改名为`LwVM`
- 构造方式：原生随机数，**在首次安装操作系统或被用户擦除设备时创建（ToDO：？）**
- 加密方式：decrypt(EMF key, key 0x89B)
- 储存位置：NAND闪存的可擦除区域

### Dkey：设备密钥，Device key

- 功能描述：就是`NSProtectionNone` 类密钥的封装密钥，主要用于远程数据擦除
- 构造方式：原生随机数
- 加密方式：decrypt(Dkey, key 0x835)
- 储存位置：NAND闪存的可擦除区域

> 闪存Flash的特点是wear-leveling，删除数据很困难，但加密了就容易多了，直接删除密钥就OK

### BAG1: system bag

- 功能描述：系统密钥包（`/private/var/keybags/systembag.kb`）的文件级密钥，并包含一个初始向量IV
- 构造方式：原生随机数
- 加密方式：**不加密**。系统密钥包实际是一个数据文件，保存的Class Key还有一层加密
- 储存位置：NAND闪存的可擦除区域

## 二. 安全隔区有那些内置的核心密钥？

![密钥关系图](keys.png)

### UID & GID

UID 密钥：嵌入在应用处理器 AES 引擎中的硬件密钥，对于每个设备都是唯一的。该密钥可以被 CPU 使用，但不能被 CPU 读取。可以从引导加载程序和内核模式使用。也可以通过修补 IOAESAccelerator 从用户态使用。如Apple iOS 安全文档中所述：

“设备的唯一 ID (UID) 和设备组 ID (GID) 是在制造过程中融合到应用处理器中的 AES 256 位密钥。”
“没有软件或固件可以直接读取它们；他们只能看到使用它们执行的加密或解密操作的结果”
“UID 对每台设备都是唯一的，Apple 或其任何供应商都不会记录。”
“UID 与设备上的任何其他标识符无关。”
“一个 256 位 AES 密钥，在制造时已刻录到每个处理器中。”
“它不能被固件或软件读取，只能由处理器的硬件 AES 引擎使用。”
“为了获得真正的密钥，攻击者必须对处理器的芯片进行高度复杂且昂贵的物理攻击。”
“UID 与设备上的任何其他标识符无关，包括但不限于 UDID。”
UIDPlus 密钥：iOS 5 内核引用的新硬件密钥，在 iPhone 4S 上未使用，从 iPad 2 开始使用？

`UDID = SHA1(Serial Number + ECID + LOWERCASE (WiFi Address) + LOWERCASE(Bluetooth Address))`

### 派生密钥

仅基于 UID 或 GID，安全隔区启动后在内存中临时计算，无需持久化存储。

- Key 0x835 = AES(UID, 01010101010101010101010101010101)；保护`Class key`
- Key 0x836 = AES(UID, 00E5A0E6526FAE66C5C1C6D4F16D6180)
- Key 0x837 = AES(GID, 345A2D6C5050D058780DA431F0710E15)
- Key 0x838 = AES(UID, 8C8318A27D7F030717D2B8FC5514F8E1)
- Key 0x89B = AES(UID, 183e99676bb03c546fa468f51c0cbd49)；保护`EMF key`

### Passcode Key

Passcode Key是用戶輸入的passcode結合系統硬件的加密引擎以及PBKDF2(Password-Based Key Derivation Function)算法生成的。PBKDF2的基本原理是通過一個偽隨機函數，把明文和一個鹽值及加密重複次數作為輸入參數，然後重複進行運算，並最終產生密鑰。重複運算的會使得暴力破解的成本變得很高，而硬件key及鹽值的添加基本上斷絕了通過“彩虹表”攻擊的可能。

由於硬件加密引擎的Key無法提取，所以只能在目標的機器上運行暴力破解程序進行破解，假設用戶的密碼設置的足夠複雜的話，那麼破解的周期就會變得非常久。

KDF = Deriving Key from Password

密码密钥：使用 Apple 自定义派生函数 (Tangling) 从用户密码或托管密钥包 BagKey 计算得出。用于从系统/托管密钥包中解开类密钥。解开钥匙包钥匙后立即从内存中删除。

### Filesystem Key：文件系统密钥

系统会使用文件系统密钥(the file system key)解密文件的元数据， 以显露出封装的文件独有密钥和表示它受哪个类保护的记号
用于加密每个文件的元数据的密钥， 包括其类密钥。 存储在可擦除存储器中， 用于实现快速擦除， 并非用于保密目的。
加密的文件系统密钥还会使用储存在可擦除存储器中的“可擦除密钥”( an “effaceable key”) 封装，或者使用受安全隔区反重放机制保护的媒介密钥封装密钥（a media key-wrapping key）进行封装。
> the Secure Storage Component’s unique cryptographic key
此密钥不会提供数据的额外机密性。 相反，它可以根据需要快速抹掉。

> 数据宗卷文件系统中所有文件的元数据都使用随机宗卷密钥(a random volume key)进行加密， 该密钥在首次安装操作系统或用户擦除设备时创建。
> 此密钥由密钥封装密钥(a key wrapping key)加密和封装， 密钥封装密钥由安全隔区长期储存， 只在安全隔区中可见。 每次用户抹掉设备时， 它都会发生变化。
> 在 A9 （及后续型号） SoC 上， 安全隔区依靠由反重放系统支持的熵来实现可擦除性， 以及保护其他资源中的密钥封装密钥。 有关更多信息， 请参阅安全非易失性存储器。

文件系统密钥用于保护GPT分区表（GUID Partition Table）和系统分区（System partition）。
Todo： 也被称为 NAND 密钥？
文件系统密钥（f65dae950e906c42b254cc58fc78eece）

### Metadata Key：元数据密钥

元数据密钥(92a742ab08c969bf006c9412d3cc79a5)：加密 NAND 元数据（vfl/ftl 上下文和索引页）
![文件保险箱](FileVault.png)

## 三、系统密钥包（System Bag）的工作原理是什么？

- 文件内容被加密，每个文件都有独立的文件密钥`Per-file Key`
- `Per-file Key`被`Class Key`加密，存储位置：该文件的元数据 Meatadata 的`cprotect`字段

File is encrypted with a File Key
File Key encrypted with Class Key
Class Key encrypted with Passcode Key Passcode key derived from:
    • •
UID, 0x835, Passcode
Keybag encrypted with Bag Key Entire disk encrypted with EMF Key 
EMF key encrypted using 0x89b 
0x89b and 0x835 derived from UID



以 UID 为基础衍生出多个密钥，其中：

- `key 0x89B`:
- ，會衍生出多隻金鑰，其中兩隻金鑰、0x89B與0x835會用於Device Keybag上。0x89B功能同FDE，會透過EMF金鑰針對整個File System/Partition進行加密保護。而Passcode則是用戶自己產生，從iOS 9之後，預設的Passcode長度為6個數字，而0x835 Key與Passcode Key(從Passcode透過加密運算產生)會針對所有的Class Key進行加密(除了Class 4 Key外)。

data partition 每个文件创建时，Data Protection会创建一个256-bit key (“per-file” key) ，然后使用这个密钥借助硬件 AES engine。加密模式通常是AES CBC mode（不懂的看密码学）. 既然是CBC自然需要IV，IV怎么来， 用文件的块偏移计算LFSR（ linear feedback shift register (LFSR) calculated with the block offset into the file）, IIV也是加密存储，encrypted with the SHA-1 hash of the per-file key.

per-file key是实际用来加密文件的，那它也得被保护啊。这就得分好几种条件了。
然后，加密的 per-file key is stored in the file’s metadata。.

然后file’s metadata又被加密了，The metadata of all files in the file system are encrypted with a random key, which is created when iOS is first installed or when the device is wiped by a user. The file system key is stored in Effaceable Storage. 其实这个加密价值不大。主要是为了销毁密钥方便。ios界面上操作 “Erase all content and settings” option,就是干掉了这个密钥。最终无法获得加密的per-file key。per-file key到底被谁加密，环境很多。根据不同的环境使用不用的 class key。 class key 又被 hardware UID 保护，有些情况下是通过user’s passcode. 如果你的数据相关联pin，那就得靠他。这非常重要！！！

多层密钥架构还有一个好处，就是对加密的内容更换密钥，无需重新加解密数据，那样效率很低，而只需更换加密key。

有密碼保護與無密碼保護的iTunes備份最大的差異是在Keybag是否被UID Key加密的狀態下儲存於iTunes備份擋內。若iTunes備份檔沒有密碼保護，Keybag會被UID的金鑰加密後儲存於備份檔內(Keychain也是)，這代表這個備份檔案是無法恢復到其他的iOS裝置，僅能恢復到原有備份出來的裝置(因為每個iOS的UID皆不同)。若iTunes備份檔有密碼保護，則Keybag會用iTunes設定的密碼採PBKDF2加密函式反覆運算1千萬次來保護 Keybag ，因Keybag未與UID綁定，當升級新的iOS裝置時就必須採用密碼保護的iTunes備份檔才可進行資料移轉。

有一个特定的储物柜叫做BAGI，它包含一个加密密钥，用来加密所谓的系统密钥包。密钥包包含许多加密“类密钥”，最终保护文件系统中的文件;它们在不同的时间被锁定和解锁，这取决于用户的活动。这让开发人员可以选择在设备被锁定时是否锁定文件，或者在输入密码后保持解锁，等等。文件系统上的每个文件都有自己的随机文件密钥，该密钥使用密钥包中的类密钥进行加密。密钥包的密钥是由BAGI储物柜中的密钥和用户的PIN的组合加密的。

NAND还有另一个储物柜(苹果称之为4类键，我们称之为Dkey)。Dkey不使用用户PIN进行加密，在之前的iOS版本(<8)中，它被用作没有特别使用“数据保护”保护的任何文件的加密基础。根据设计，当时的大多数文件系统都使用Dkey而不是类键。因为密码中没有涉及PIN(就像密钥包中的类密钥一样)，任何具有根级别访问权限的人(比如Apple)都可以轻松地打开Dkey锁，从而解密使用它进行加密的文件系统的绝大多数。在iOS 8之前，只有那些明确启用了数据保护功能的文件才使用PIN保护，这并不包括大多数存储个人数据的苹果文件。在iOS 8中，苹果最终将文件系统的其余部分从Dkey储物柜中取出，现在几乎整个文件系统都在使用密钥包中的类密钥，这些密钥由用户的PIN进行保护。硬件加速的AES加密功能允许对整个硬盘进行非常快速的加密和解密，这在技术上从3GS开始就成为可能，然而，由于没有任何有效的原因(除了设计决定)，苹果决定在iOS 8之前不正确地加密文件系统。




![基本结构](arch1.png)
![简化架构](arch0.jpeg)


``` js
HEADER
  VERS = 4
  TYPE = 0
  UUID = 32 HEX
  HMCK = 80 HEX
  WRAP = 1
  SALT = 40 Hex
  ITER = 50000
  TKMT = 0
  SART = 98
  UUID = 32 HEX
KEYS
  0:
    CLAS = 1
    WRAP = 3
    KTYP = 0
    WPKY = 80 HEX
    UUID = 32 HEX

... up to
  9:
```

Hierarchical File System,分层文件系统

---

## 参考文献

### 精品资料

- [面向开发人员的实用密码学 - Svetlin Nako](https://cryptobook.nakov.com/mac-and-key-derivation/kdf-deriving-key-from-password)
- [iPhone 4 取证工具包 - Github](https://github.com/nabla-c0d3/iphone-dataprotection)

### 文档下载

- [iOS加密技术高级分析](2016-BSidesROC-iOSCrypto.pdf)
- [iOS Encryption Systems - 奥地利格拉茨技术大学论文](iOS_Encryption_Systems.pdf)
- [iOS 5的数据保护技术分析](0721C6_Andrey.Belenko_Evolution.of.iOS.Data.Protection.pdf)
- [iPhone 数据保护技术分析 - Sogeti](iPhone_Data_Protection_in_Depth.pdf)
- [iPhone 数据保护技术 - ELComSoft](OWASP_BeNeLux_Day_2011_-_A._Belenko_-_Overcoming_iOS_Data_Protection.pdf)
- [iOS后门分析](iOS_backdoor.pdf)

### 技术分析

- [ios安全团队对ios安全的认识](https://bbs.pediy.com/thread-174935.htm)
- [iOS资料保护机制简介](https://www.kaotenforensic.com/ios/ios-data-protection/)
- [苹果的锁屏密码到底有多难破？ - 头条](https://toutiao.io/posts/324007/app_preview)
- [超越FBI NSA, iPhone 5c 以物理NAND備份法破解iOS密碼](https://www.osslab.com.tw/iphone-5c-nand/)
- [Jonathan Zdziarski 对iOS文件系统的论述](https://www.theiphonewiki.com/wiki/File_System_Crypto)
- [FBI vs Apple：FBI是幸运的 - 盘古团队](https://www.leiphone.com/category/zhuanlan/4dO3QQ178rkZ3mo5.html)
- [iOS 破解分析 - 乌云](https://paper.seebug.org/papers/Archive/drops2/%E3%80%8AiOS%E5%BA%94%E7%94%A8%E5%AE%89%E5%85%A8%E6%94%BB%E9%98%B2%E5%AE%9E%E6%88%98%E3%80%8B%E7%AC%AC%E5%85%AD%E7%AB%A0%EF%BC%9A%E6%97%A0%E6%B3%95%E9%94%80%E6%AF%81%E7%9A%84%E6%96%87%E4%BB%B6.html)
- [OS 安全攻防之敏感数据保护](https://heyonly.github.io/2017/07/02/iOS-%E5%AE%89%E5%85%A8%E6%94%BB%E9%98%B2%E4%B9%8B%E6%95%8F%E6%84%9F%E6%95%B0%E6%8D%AE%E4%BF%9D%E6%8A%A4/)
- [iOS backdoor](iOS_backdoor.pdf)
- [FBE 文件级加密原理 - Android官方](https://source.android.com/docs/security/features/encryption/file-based?hl=zh-cn)
- [你的安卓手机究竟是FDE加密还是FBE加密？](https://page.om.qq.com/page/O3yauEIx2l-9WrUkHgQgRUBw0)
- [Android 檔案系統加密機制](https://www.kaotenforensic.com/android/android_encryption/)
- [Android 系統基本架構 - 開機流程與分區說明](https://www.kaotenforensic.com/android/booting-partitions/)
- [iOS 数据保护基础知识](https://www.pmbonneau.com/multiboot/dataprotection_basics.php)
- [拆解 iPhone 的黑客指南（第 3 部分](http://securityhorror.blogspot.com/2013/09/the-hackers-guide-to-dismantling-iphone_5697.html)