---
title: Apple数据保护（Data Protection）技术概览
date: 2022-10-05 21:19:00
tags:
---

## 用户数据保护

![Key](FBE.jpg)
![Key](KEY-ARCH.jpg)

## 4种保护类型 Class

### A类：完全保护，NSFileProtectionComplete

- 该类密钥通过从用户密码和设备 UID 派生的密钥得到保护。 用户锁定设备后不久 （如果 “需要密码” 设置为 “立即”， 则为 10 秒钟）， 解密的类密钥会被丢弃， 此类的所有数据都无法访问， 除非用户再次输入密码或使用触控 ID 或面容 ID 解锁 （登录） 设备。
- 在 macOS 中， 上一个用户退出登录不久后， 解密的类密钥会被丢弃， 此类的所有数据都无法访问， 直到某位用户再次输入密码或使用触控 ID 登录设备。

### B类：未打开文件保护，NSFileProtectionCompleteUnlessOpen

- 设备锁定或用户退出登录时， 可能需要写入部分文件。 如邮件附件在后台下载。 此行为通过使用非对称椭圆曲线加密技术 （基于 Curve25519 的 ECDH） 实现。 普通的文件独有密钥通过使用一次性迪菲-赫尔曼密钥交换协议 （One-Pass Diffie-Hellman Key Agreement， 如 NIST SP 800-56A 中所述） 派生的密钥进行保护。
- 该协议的临时公钥与封装的文件独有密钥一起储存。 KDF 是串联密钥导出函数 (Approved Alternative 1)， 如 NIST SP 800-56A 中 5.8.1 所述。 AlgorithmID 已忽略。 PartyUInfo 和 PartyVInfo 分别是临时公钥和静态公钥。 SHA256 被用作哈希函数。 一旦文件关闭， 文件独有密钥就会从内存中擦除。 要再次打开该文件， 系统会使用 “未打开文件的保护” 类的私钥和文件的临时公钥重新创建共享密钥， 用来解开文件独有密钥的封装， 然后用文件独有密钥来解密文件。
- 在 macOS 中， 只要系统上的任何用户已登录或认证即可访问 NSFileProtectionCompleteUnlessOpen 的私有部分。

### C类：首次用户认证前保护，NSFileProtectionCompleteUntilFirstUserAuthentication

- 此类和 “全面保护” 类的行为方式相同， 只不过在设备锁定或用户退出登录时已解密的类密钥不会从内存中删除。 此类中的保护与桌面电脑全宗卷加密有类似的属性， 可防止数据受到涉及重新启动的攻击。 这是未分配给数据保护类的所有第三方 App 数据的默认类。
- 在 macOS 中， 此类的作用类似于文件保险箱， 且使用只要宗卷装载即可访问的宗卷密钥。

### D类：无保护，NSFileProtectionNone

- 此类密钥仅受 UID 的保护， 并且存储在可擦除存储器中。 由于解密该类中的文件所需的所有密钥都储存在设备上， 因此采用该类加密的唯一好处就是可以进行快速远程擦除。 即使未向文件分配数据保护类， 此文件仍会以加密形式储存 （就像 iOS 和 iPadOS 设备上的所有数据那样）。
- macOS 不支持该类。


|Availability |File Data Protection |Keychain Data Protection |
|-|-|-|
|When unlocked |NSFileProtectionComplete |kSecAttrAccessibleWhenUnlocked|完全保护|
|While locked |NSFileProtectionCompleteUnlessOpen |N/A |未打开文件保护|
|After first unlock |NSFileProtectionCompleteUntilFirstUserAuthentication |kSecAttrAccessibleAfterFirstUnlock |首次认证前保护|
|Always |NSFileProtectionNone |kSecAttrAccessibleAlways|不保护|
|Passcode enabled |N/A |kSecAttrAccessible WhenPasscodeSetThisDeviceOnly|

## 5钟密钥包

- User keyBag: 用户
- Device keyBag: 设备
- Backup keyBag: 备份
- Escrow keybag: 托管
- iCloud Backup keybag: 云备份。

sepOS 使用 UID 来保护设备特定的密钥。 有了 UID， 就可以通过加密方式将数据与特定设备捆绑起来。 例如， 用于保护文件系统的密钥层级就包括 UID， 因此如果将内置 SSD 储存设备从一台设备整个移至另一台设备， 文件将不可访问。 其他受保护的设备特定密钥包括触控 ID 或面容 ID 数据。 在 Mac 上， 只有链接到AES 引擎的完全内置储存设备才会获得这个级别的加密。 例如， 通过 USB 连接的外置储存设备或者添加到2019 年款 Mac Pro 的基于 PCIe 的储存设备都无法获得这种方式的加密。

---

## 参考文献

- [iOS 安全设计解析（二）：数据加密和保护](https://blog.blupig.net/ios-security-encryption)
- [iOS资料保护机制简介](https://www.kaotenforensic.com/ios/ios-data-protection/)
