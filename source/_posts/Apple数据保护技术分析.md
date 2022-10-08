---
title: Apple数据保护技术分析
date: 2022-10-08 20:05:45
tags:
---

花了好长时间，才明白Apple公司数据保护（Data Protection）的核心。

根据Apple安全白皮书的官方文档，数据保护实际上有两条路径，一是面向智能终端的 iOS 发展来的 Data Protection，二是面向个人电脑的 MacOS 发展来的数据保险箱，由于依赖Intel CPU，数据安全就是各种打补丁。M1 发布后两条路径开始融合，Secure Encalve 和 AFPS 成为融合的基石。
任何技术都有路径依赖，虽然 HFS+ 被认为是业界最烂的文件系统，但其许多重要的核心功能仍然需要 AFPS 继承下来，当然也为后续发展打下基础。

## 一、基础原理

![简化架构](arch0.jpeg)

- 每个文件创建时会生成一个`Per-file Key`，在文件写入闪存时通过硬件用`AES-XTS`算法加密
- `Per-file Key`存储在文件元数据 Metadata 中，被`Class Key`和`Filesystem Key`加密
- `Class Key`和`Filesystem Key`都被硬件密钥`UID`保护
- `Class Key`还被用户密码`Passcode`保护

实际上，Apple就是在 HFS+ 文件系统的 inode 中增加了一个`cprotect` 字段：

- 如果数据保护级别是 A / B / C，存储以`Class Key`加密的`Per-file Key`
- 如果数据保护级别是 D，存储加密后的`DKey`(Device Key，正好也是D)
- 如果设置为`discarded`，则意味着密码失效，文件就永远无法打开，也是数据被抹去了...

## 二、物理架构

![基本结构](arch3.jpg)

### 1. 文件系统密钥、类密钥的存储位置在哪里？

CPU都是通用的，这些关键密钥都是个性化数据，必须找个存储资源！
秘密就藏在 NAND 闪存的1号扇区，Apple 将它作为隐藏的可安全擦除区域(Effaceable Storage)，就是安全隔区的技术架构中的`Secure nonvolatile storage`。当然，Apple官方文档说安全隔区通过专用 I2C 总线连接，但 NAND 闪存不太可能再增加一个物理接口，也许是复用吧。

> Effaceable Storage：accesses the underlying storage technology (for example, NAND) to directly address and erase a small number of blocks at a very low level.

在这个关键的区域，存储了许多关键数据：

- EMF：文件系统主密钥，file-system master encryption key。其实是文件系统日志的加密密钥（记得磁盘格式化时Macos日志式文件系统！）
    > 注意！EMF key存储时是明文，因此任何人在没有用户密码的情况下很容易解密HFS日志！
- DKey：设备密钥，Device Key，来自于 UID 派生。
    保护类型 D 号称无保护，但实际上通过 DKey 获得 UID 保护，。主要用于远程数据擦除。
    > flash的特点是wear-leveling，删除数据很困难，但加密了就容易多了，直接删除密钥就OK
- BAGI：KeyBag的密钥？
- 设备密钥包：就是安全白皮书中说的 Device keyBag，也就是存储`Class Key`的数据文件，安全隔区正常启动并成功解密后，将其内容加载至内存。

### 2. 核心密钥的派生关系是什么？

以 UID 为基础衍生出多个密钥，其中：

- `key 0x89B`:
- ，會衍生出多隻金鑰，其中兩隻金鑰、0x89B與0x835會用於Device Keybag上。0x89B功能同FDE，會透過EMF金鑰針對整個File System/Partition進行加密保護。而Passcode則是用戶自己產生，從iOS 9之後，預設的Passcode長度為6個數字，而0x835 Key與Passcode Key(從Passcode透過加密運算產生)會針對所有的Class Key進行加密(除了Class 4 Key外)。

`Key 0x835 = AES_encrypt (UID, 0101..01)`。。也是UID直接计算出来的，也是和设备绑定的。

data partition 每个文件创建时，Data Protection会创建一个256-bit key (“per-file” key) ，然后使用这个密钥借助硬件 AES engine。加密模式通常是AES CBC mode（不懂的看密码学）. 既然是CBC自然需要IV，IV怎么来， 用文件的块偏移计算LFSR（ linear feedback shift register (LFSR) calculated with the block offset into the file）, IIV也是加密存储，encrypted with the SHA-1 hash of the per-file key.

per-file key是实际用来加密文件的，那它也得被保护啊。这就得分好几种条件了。
然后，加密的 per-file key is stored in the file’s metadata。.

然后file’s metadata又被加密了，The metadata of all files in the file system are encrypted with a random key, which is created when iOS is first installed or when the device is wiped by a user. The file system key is stored in Effaceable Storage. 其实这个加密价值不大。主要是为了销毁密钥方便。ios界面上操作 “Erase all content and settings” option,就是干掉了这个密钥。最终无法获得加密的per-file key。per-file key到底被谁加密，环境很多。根据不同的环境使用不用的 class key。 class key 又被 hardware UID 保护，有些情况下是通过user’s passcode. 如果你的数据相关联pin，那就得靠他。这非常重要！！！

多层密钥架构还有一个好处，就是对加密的内容更换密钥，无需重新加解密数据，那样效率很低，而只需更换加密key。




![基本结构](arch1.png)
![更细架构](arch2.png)


---

## 参考文献

- [ios安全团队对ios安全的认识](https://bbs.pediy.com/thread-174935.htm)
- [iOS资料保护机制简介](https://www.kaotenforensic.com/ios/ios-data-protection/)
- [苹果的锁屏密码到底有多难破？ - 头条](https://toutiao.io/posts/324007/app_preview)
- [超越FBI NSA, iPhone 5c 以物理NAND備份法破解iOS密碼](https://www.osslab.com.tw/iphone-5c-nand/)
- [Jonathan Zdziarski 对iOS文件系统的论述](https://www.theiphonewiki.com/wiki/File_System_Crypto)
- [FBI vs Apple：FBI是幸运的 - 盘古团队](https://www.leiphone.com/category/zhuanlan/4dO3QQ178rkZ3mo5.html)
- [iOS 破解分析 - 乌云](https://paper.seebug.org/papers/Archive/drops2/%E3%80%8AiOS%E5%BA%94%E7%94%A8%E5%AE%89%E5%85%A8%E6%94%BB%E9%98%B2%E5%AE%9E%E6%88%98%E3%80%8B%E7%AC%AC%E5%85%AD%E7%AB%A0%EF%BC%9A%E6%97%A0%E6%B3%95%E9%94%80%E6%AF%81%E7%9A%84%E6%96%87%E4%BB%B6.html)
- [OS 安全攻防之敏感数据保护](https://heyonly.github.io/2017/07/02/iOS-%E5%AE%89%E5%85%A8%E6%94%BB%E9%98%B2%E4%B9%8B%E6%95%8F%E6%84%9F%E6%95%B0%E6%8D%AE%E4%BF%9D%E6%8A%A4/)
- [iOS backdoor](iOS_backdoor.pdf)
