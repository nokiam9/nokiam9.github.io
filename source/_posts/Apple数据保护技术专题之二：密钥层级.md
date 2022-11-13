---
title: Apple数据保护技术专题之二：密钥层级
date: 2022-11-13 14:34:19
tags:
---


## 四、Dkey 密钥的演进分析

根据 iOS 的初期设计，`Class D`是所有用户数据文件的默认类型，其类密钥就是 DKey，即大多数文件的`per-file key`都通过 Dkey 进行封装。
此时 DKey 的存储路径就是`systembag.kb`文件的 `Class 4` 字段，当然受到了基于 UID 的 Key 0x835 的保护。

由于 DKey 直接保存在一个普通的数据文件中，任何具有 root 级别访问权限的用户都可以轻松地获得并展开暴力破解，这显然是非常危险的。
从 iOS 5 开始，苹果公司将 DKey 从设备密钥包中取出，存储路径转移到安全隔区专用的`Effaceable Storage`之中。
从 iOS 8 开始，为了应对 GrayKey 密码破解设备破解苹果公司又开发了


由于 DKey 仅受 UID 的保护，，从而解密绝大多数文件
> iOS 8 之后，苹果公司将文件系统的默认类型调整为`Class C`，目的就是将类密钥纳入 Passcode 的保护范围，因此目前 NSFileProtectionNone 类型的文件已经大为减少，这些文件在设备开机后就一直保持可访问状态

## Effaceable lockers

- EMF!
    • Data partition encryption key, encrypted with key 0x89B
    • Format: length (0x20) + AES(key89B, emfkey)
- Dkey
    • NSProtectionNone class key, wrapped with key 0x835
    • Format: AESWRAP(key835, Dkey)
- BAG1
    • System keybag payload key
    • Format : magic (BAG1) + IV + Key
    • Read from userland by keybagd to decrypt systembag.kb
    • Erased at each passcode change to prevent attacks on previous keybag


AppleEffaceableStorage IOKit userland interface
Selector Description Comment
0 getCapacity 960 bytes
1 getBytes requires PE_i_can_has_debugger
2 setBytes requires PE_i_can_has_debugger
3 isFormatted
4 format
5 getLocker input : locker tag, output : data
6 setLocker input : locker tag, data
7 effaceLocker scalar input : locker tag
8 lockerSpace ?


## 一、安全隔区的内置核心密钥

![Arch0](arch0.jpg)

### 3. UID 派生密钥

- 根据不同用途使用各自的派生密钥，而不是直接使用 UID，可以有效减少 UID 被泄露的风险
- 安全隔区启动时，将从 UID 派生出多个硬件密钥，加密因子是不同的**固定盐**
- 这些派生的硬件密钥都在内存中，无需持久化存储

以下是一些常用的派生密钥：

- Key 0x835 = AES(UID, 01010101010101010101010101010101)；保护`Class key`，也被称为`device key`
- Key 0x836 = AES(UID, 00E5A0E6526FAE66C5C1C6D4F16D6180)
- Key 0x837 = AES(GID, 345A2D6C5050D058780DA431F0710E15)
- Key 0x838 = AES(UID, 8C8318A27D7F030717D2B8FC5514F8E1)
- Key 0x89B = AES(UID, 183e99676bb03c546fa468f51c0cbd49)；保护`EMF key`

### 1. UID & GID

- UID 是一个 AES 256 位密钥，在 SOC 制造过程中写入一次性的**熔丝**
- 每个设备的 UID 唯一且无法更改，Apple 或其任何供应商都不会记录 UID
- UID 不能被固件或软件读取，只能由处理器的硬件 AES 引擎使用
- UID 与设备上的任何其他标识符无关，包括但不限于 UDID

> UDID（Unique Device Identifier）是苹果 iOS 设备的唯一识别码，它由40个字符的字母和数字组成，利用 UDID 可以识别移动设备和跟踪用户行为
> `UDID = SHA1(Serial Number + ECID + LOWERCASE (WiFi Address) + LOWERCASE(Bluetooth Address))`

### 3. Class Key 的使用

用户开机成功后，类密钥就保留在内存中，后续将根据不同事件触发相应的处理流程。

#### 主动锁屏 & 超时锁屏

• FileProtectionComplete key removed from RAM
• All Complete protection files now unreadable
• Other keys remain present
• Allows connection to Wi-Fi 
• Lets you see contact information when phone rings
• [I once found an edge case where this doesn’t happen…]

- 设备锁屏 10 秒后，Class key A 将从内存中删除，此时无法访问 `NSFileProtectionComplete` 保护类型的文件（例如照片、记事本等数据）
- `NSFileProtectionCompleteUntilFirstUserAuthentication`类型的文件仍然可以访问（这是所有第三方 APP 数据文件的默认类型），因为Class key C 在内存中
- `NSFileProtectionCompleteUnlessOpen` 是为了解决有些文件可能需要在锁屏后继续写入的需求，例如后台下载邮件附件等，
    此行为通过使用非对称椭圆曲线加密技术实现，将临时公钥与封装的文件独有密钥一起储存。一旦文件关闭，`per-file key`就会从内存中擦除。
    要再次打开该文件，系统会使用私钥和文件的临时公钥重新创建共享密钥，用来解开`per-file key！`的封装， 然后用`per-file key`来解密文件

#### 用户重设 passcode

• The system keybag is duplicated
• Class keys wrapped using new passcode key (encrypted 
with 0x835 key, wrapped with passcode)
• New BAG key created and stored in effaceable storage
• Old BAG key thrown away
• New keybag encrypted with BAG key

#### 用户擦除 NAND 数据

• mobile_obliterator daemon
• Erase DKey by calling MKBDeviceObliterateClassDKey
• Erase EMF key by calling selector 0x14C39 in EffacingMediaFilter service
• Reformat data partition
• Generate new system keybag
• High level of confidence that erased data cannot be recovered

Effaceable storage is wiped, destroying:
• DKey: All “File protection: none” files are unreadable
• Bag key: All other class keys are unreadable
• EMF key: Can’t decrypt the filesystem anyway

### 设备重启

• File Protection Complete key lost from RAM
• Complete until First Authentication key also lost
• Only “File Protection: None” files are readable
• And then only by the OS on the device
• Because FDE