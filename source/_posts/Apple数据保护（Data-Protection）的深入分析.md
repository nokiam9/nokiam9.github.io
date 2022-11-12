---
title: Apple数据保护（Data Protection）的深入分析
date: 2022-10-30 16:50:10
tags:
---

长期以来 Apple 公司有两条数据安全的技术演进路线，iPhone 和 iPad 等智能终端设备使用称为**数据保护（Data Protection）**的文件加密方法，基于Intel的Mac设备通过**文件保险箱（FileVault）**的宗卷加密技术，造成这种问题的原因有：

1. MacOS 是基于 FreeBSD 发展而来，基础架构是一个多用户的操作系统，但 iPhone 等作为个人化的终端设备，无需支持多用户，但非常强调个人数据的隐私保护，产品定位导致业务需求存在明显差异；
2. iOS 是在 MacOS 的基础上改进而来，但受限于 MacOS 的底层技术架构，尤其是被称为业界最烂的文件系统 - **HFS+（Hierarchical File System）**，许多急需的新功能只能通过纯软件的方法强行实现；
3. iPhone 很早就采用自研芯片，非常有利于制定软硬件一体化的解决方案，而 MacOS 受到 Intel CPU 的限制，兼容性要求导致技术演进存在障碍。

> 对比 Android 系统，早期采用的**FDE（Full-Disk Encryption）**模式也是一种基于 Volume 的全盘加密技术；从 Android 7 开始，推出 **FBE（File-based Encryption）**模式可以支持文件级别的数据加密，但技术可靠性一直问题多多，直到 Android 13 才基本成熟并取消 FDE 模式的支持，这也从另一个角度说明数据保护技术的复杂性。

随着 M1 自研芯片的推出，两条技术路线正在逐步融合，搭载 Apple 芯片的 Mac 设备已经使用两者的混合模型，相信未来 FileVault 技术将逐步退出，因此本文主要研究Data Protection 技术。根据Apple 官方文档，其主要设计目标包括：

- 在硬件被改动的情况下也能保护数据（替换组件/直接读取闪存等）
- 对抗离线攻击（物理方式获取闪存内的数据后用更强大的计算设备进行破解）
- 对不同的数据提供不同级别的保护（锁屏之后有些数据要保护，有些数据还需要访问）
- 需要时能够快速安全清除所有数据
- 考虑性能和功耗

## 一、安全隔区的内置核心密钥

![Arch0](arch0.jpg)

### 1. UID & GID

- UID 是一个 AES 256 位密钥，在 SOC 制造过程中写入一次性的**熔丝**
- 每个设备的 UID 唯一且无法更改，Apple 或其任何供应商都不会记录 UID
- UID 不能被固件或软件读取，只能由处理器的硬件 AES 引擎使用
- UID 与设备上的任何其他标识符无关，包括但不限于 UDID

> UDID（Unique Device Identifier）是苹果 iOS 设备的唯一识别码，它由40个字符的字母和数字组成，利用 UDID 可以识别移动设备和跟踪用户行为
> `UDID = SHA1(Serial Number + ECID + LOWERCASE (WiFi Address) + LOWERCASE(Bluetooth Address))`

### 2. Passcode & Passcode Key

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

## 二、安全存储组件

### 1. EMF

### 2. Dkey

### 3. BAG1

### 4. 计数器加密箱

## 三、密钥包 - Keybag

## 四、钥匙包 - KeyChain

根据[https://opensource.apple.com/source/xnu/xnu-1699.32.7/bsd/hfs/hfs_cnode.h](https://opensource.apple.com/source/xnu/xnu-1699.32.7/bsd/hfs/hfs_cnode.h)

``` c
/*
 * The cnode is used to represent each active (or recently active)
 * file or directory in the HFS filesystem.
 *
 * Reading or writing any of these fields requires holding c_lock.
 */
struct cnode {
    lck_rw_t        c_rwlock;               /* cnode's lock */
    void            *c_lockowner;           /* cnode's lock owner (exclusive case only) */
    lck_rw_t        c_truncatelock;         /* protects file from truncation during read/write */
    void            *c_truncatelockowner;   /* truncate lock owner (exclusive case only) */
    LIST_ENTRY(cnode)   c_hash;             /* cnode's hash chain */
    u_int32_t       c_flag;                 /* cnode's runtime flags */
    u_int32_t       c_hflag;                /* cnode's flags for maintaining hash - protected by global hash lock */
    struct vnode    *c_vp;                  /* vnode for data fork or dir */
    struct vnode    *c_rsrc_vp;             /* vnode for resource fork */
    struct dquot    *c_dquot[MAXQUOTAS];    /* cnode's quota info */
    u_int32_t       c_childhint;            /* catalog hint for children (small dirs only) */
    u_int32_t       c_dirthreadhint;        /* catalog hint for directory's thread rec */
    struct cat_desc c_desc;                 /* cnode's descriptor */
    struct cat_attr c_attr;                 /* cnode's attributes */
    TAILQ_HEAD(hfs_originhead, linkorigin) c_originlist;  /* hardlink origin cache */
    TAILQ_HEAD(hfs_hinthead, directoryhint) c_hintlist;  /* readdir directory hint list */
    int16_t         c_dirhinttag;           /* directory hint tag */
    union {
        int16_t     cu_dirhintcnt;          /* directory hint count */
        int16_t     cu_syslockcount;        /* system file use only */
    } c_union;
    u_int32_t       c_dirchangecnt;         /* changes each insert/delete (in-core only) */
    struct filefork *c_datafork;            /* cnode's data fork */
    struct filefork *c_rsrcfork;            /* cnode's rsrc fork */
    atomicflag_t    c_touch_acctime;
    atomicflag_t    c_touch_chgtime;
    atomicflag_t    c_touch_modtime;
#if HFS_COMPRESSION
    decmpfs_cnode  *c_decmp;
#endif /* HFS_COMPRESSION */
#if CONFIG_PROTECT
    cprotect_t      c_cpentry;              /* content protection data */
#endif
};
```

根据[https://opensource.apple.com/source/xnu/xnu-1699.32.7/bsd/sys/cprotect.h.auto.html](https://opensource.apple.com/source/xnu/xnu-1699.32.7/bsd/sys/cprotect.h.auto.html)

``` c
typedef struct cprotect *cprotect_t;

/* 
 * Runtime-only structure containing the content protection status 
 * for the given file.  This is contained within the cnode 
 */
struct cprotect {
    uint8_t     cp_cache_key[CP_KEYSIZE];
    uint8_t     cp_persistent_key[CP_WRAPPEDKEYSIZE];
    uint32_t    cp_flags;
    uint32_t    cp_pclass;
};

/*
 * On-disk structure written as the per-file EA payload 
 * All on-disk multi-byte fields for the CP XATTR must be stored
 * little-endian on-disk.  This means they must be endian swapped to
 * L.E on getxattr() and converted to LE on setxattr().	
 */
struct cp_xattr {
    u_int16_t   xattr_major_version;
    u_int16_t   xattr_minor_version;
    u_int32_t   flags;
    u_int32_t   persistent_class;
    u_int32_t   key_size;
    uint8_t     persistent_key[CP_WRAPPEDKEYSIZE];
};

/* Same is true for the root EA, all fields must be written little endian. */
struct cp_root_xattr {
    u_int16_t   major_version;
    u_int16_t   minor_version;
    u_int64_t   flags;
    u_int32_t   reserved1;
    u_int32_t   reserved2;
    u_int32_t   reserved3;
    u_int32_t   reserved4;
};
```

## 二、Device Bag 的处理方式

系统密钥包是一个加密的 `plist` 格式的二进制文件，存储了所有类密钥的数据，存储路径是 `/private/var/keybags/systembag.kb`，其全部类密钥的定义请参见：

### 1. Device Bag 的解密

早期的 iOS 系统可以使用[https://github.com/russtone/systembag.kb](https://github.com/russtone/systembag.kb) 查看 Devive Bag的信息。
注意，它其实是基一个iOS取证工具：[https://github.com/nabla-c0d3/iphone-dataprotection](https://github.com/nabla-c0d3/iphone-dataprotection)。

参考其操作步骤，Device Bag 文件解封的处理流程是：

1. 从`Effaceable Storage` 区域提取 `BAG1` ，注意其是以明文方式存储的！！！
2. 提取 key 和 iv 初始向量，对 `systembag.kb` 文件进行解密处理，获得所有 Class Key 的密文。

``` shell
$ get_bag1
iv = e859f45ec0a3ab208ec61477b74e92f0
key = 71ebb0dd387647d7b1c4d10161f5f0b622937867ffe437e41a02ccaacfe8ffb2

$ ./decrypt_systembag.py -k 71ebb0dd387647d7b1c4d10161f5f0b622937867ffe437e41a02ccaacfe8ffb2 -i e859f45ec0a3ab208ec61477b74e92f0 -o example/keybag example/systembag.kb

$ ./parse_keybag.py example/keybag
HEADER
  VERS = 4      // iOS 4.3 的版本号是 2 
  TYPE = 0      // 0 - System，1 - Backup，3 - Escrow
  UUID = cf7591b3dfc64ce8b4c36018fba96374
  HMCK = e0d8a575d2af7d15bcb26de7688d7c84eb9a4711a845b3c5d56b49c94bdc4216f165ecb4ea97ec18   // HMAC 校验值
  WRAP = 1      // 1 - UID 保护
  SALT = a358808b695d260c8a21ec801ce43db3efafecda       // 用于 Passcode KDF 的参数？
  ITER = 50000                                          // 用于 Passcode KDF 的参数？
  TKMT = 0
  SART = 98
  UUID = 9ab835423fe14b8c99b4be0ae6b066a3
KEYS
  0:
    CLAS = 1    // NSFileProtectionComplete 类
    WRAP = 3    // 3 - Passcode Key 保护
    KTYP = 0
    WPKY = 150dd562e3c6a441a879e154617d758af77553121c2b70114e32f6ad87a5819b375c724adee094ee // 该类密钥的密文
    UUID = 9d4e1c3567cc41058b3d2ee381aaa48b
  1:
    CLAS = 2    // NSFileProtectionCompleteUnlessOpen 类
    WRAP = 3
    KTYP = 1
    WPKY = 0df35185f13495d49596531f38d7114e77134a91c16915a14f2531241a78afc0ae4deeaefd2d5933
    PBKY = 0252ce8f8acc7068e4ca64cab9227035460ed5cef0661818b382e88609b1a908
  2:
    UUID = 6f537c62fd22484095c2836b227a38eb
    CLAS = 3    // NSFileProtectionCompleteUntilFirstUserAuthentication 类
    WRAP = 3
    KTYP = 0
    WPKY = 055c81e0fbc7eac3eee65e92a64c53178a95a48df6b0fa0d0ed24f3eb6b2cac9105772f6cb32c391
  3:
    UUID = 9136f182f33b46c29499cc253c94a564
    CLAS = 5    // NSFileProtectionRecovery 类，保留未启用！
    WRAP = 3
    KTYP = 0
    WPKY = c5867706ef5cc9d03b7f098a9f1f583e58397a984a17173e8d8e685fc9d2ecbc8bb3c9d76ed89c71
  4:
    UUID = 4703951420ef4ceca92d3d6e35aceb02
    CLAS = 6
    WRAP = 3
    KTYP = 0
    WPKY = acb3be303c103526aee718633a1bd946720dd128460bf54bcee99408f9ffe281a96cf352eaf5c710
  5:
    UUID = e42e8612890b48c8bee12f7c3f5510f2
    CLAS = 7
    WRAP = 3
    KTYP = 0
    WPKY = 52a6d7f05a62ccf906ce449f26325f518ad468e6d53a98bff309ac89eba088983148bada67e29a36
  6:
    UUID = 34addd5fffcb4b619d377301e49b16d5
    CLAS = 8
    WRAP = 1
    KTYP = 0
    WPKY = 9f2ddbeeb002c9897f1486244cfa5cb948cd13c23d7c480513b8be36f46a11d1
  7:
    UUID = c703f8b157b44418acbb3de1d5f35178
    CLAS = 9
    WRAP = 3
    KTYP = 0
    WPKY = a777da0779ee752fb9d7f0651ed83c7c16945af3723f30d8d82afc26eb076b99220ebffeebf1b53c
  8:
    UUID = a72aa7d9d1ea4ab6ab6a27c89961d4d4
    CLAS = 10
    WRAP = 3
    KTYP = 0
    WPKY = b7ac60ea56917249d094fd21c44728cd77621f566e7ea6538c3475a2b3d43c6061b0fbc320eb7376
  9:
    UUID = 568dfdd2f8ae4edd9ae7f22bba3c9ce2
    CLAS = 11
    WRAP = 1
    KTYP = 0
    WPKY = 44389e92846f2c7bf1294be2fcaf88153638a881197590df03e0303b1af6ac47
```

### 2. Class Key 的解封

``` py
PROTECTION_CLASSES={
    # 文件保护类型的定义
    1:"NSFileProtectionComplete",
    2:"NSFileProtectionCompleteUnlessOpen",
    3:"NSFileProtectionCompleteUntilFirstUserAuthentication",
    4:"NSFileProtectionNone",       # 已废弃，DKey = {key, iv}，改为存储在 Effaceable Storage
    5:"NSFileProtectionRecovery?",  # 未使用
    # 钥匙串类型的定义
    6: "kSecAttrAccessibleWhenUnlocked",
    7: "kSecAttrAccessibleAfterFirstUnlock",
    8: "kSecAttrAccessibleAlways",
    9: "kSecAttrAccessibleWhenUnlockedThisDeviceOnly",
    10: "kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly",
    11: "kSecAttrAccessibleAlwaysThisDeviceOnly"
}
```

对于这些 Class Key 的密文，后续是如何解密的呢？

1. 用户开机时必须手工输入 passcode
2. 安全隔区计算
    `Passcode Key = PBKDF2(passcode, salt, iter, outputLength=32)`
3. 安全隔区**逐一**解封类密钥
    `Class Key = AES_DECRYPT(Passcode Key, AES_UNWRAP(Key 0x835, Class Key!))`
4. 在所有类密钥解封完成后，与`systembag.kb`文件头部的 HMCK 进行比较，
    如果失败，可能是passcode 输入错误，或者是硬件被更换。
    如果成功，则可以使用类密钥去解封数据文件的`per-file key`

### 3. Class Key 的使用

用户开机成功后，类密钥就保留在内存中，需要根据不同事件触发相应流程

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

---

## 附录：一些有价值的资源

- [iPhone数据保护的深度分析 - iPhone Data Protection in Depth](iPhone_Data_Protection_in_Depth.pdf)
    Sogeti 是凯捷咨询集团（Capgemini）旗下的一个从事本地化技术服务的子，1967年在法国创建。该集团还包括 CAP 、GEMINI 和安永咨询等公司，是欧洲最大的IT外包服务商。

- [iOS加密技术高级分析 -  A (not-so-quick) Primer on iOS Encryption](2016-BSidesROC-iOSCrypto.pdf)
    NCC Group 成立于1999年6月，当时美国国家计算中心（the National Computing Centre）将其商业部门出售给其现有的管理团队，是一个专业的安全审计机构。

- [iOS取证技术 - iOS Forensics: Overcoming iPhone Data Protection](OWASP_BeNeLux_Day_2011_-_A._Belenko_-_Overcoming_iOS_Data_Protection.pdf)
    ElcomSoft Co. Ltd.公司于1990年在美国成立，是国际领先的数字取证工具开发公司。

- [iPhone裸闪存数据恢复的视频演讲 - iPhone raw NAND recovery and forensics](https://www.youtube.com/watch?v=5Es3wRSe3kY)
    [ibas](https://www.ibas.com/fi) 是一个芬兰的科技公司，主要业务是数据恢复和信息技术取证。

- [iOS加密系统 - iOS Encryption Systems](iOS_Encryption_Systems.pdf)
    这是一篇比较完整的技术论文，Peter Teufl 等作者来自于奥地利格拉茨技术大学。



### Apple官方文档

- [Apple 平台安全白皮书 - 2022年英文版](apple-platform-security-guide.pdf)
- [Apple 平台安全白皮书 - 2021年中文版](apple平台安全白皮书-2021中文版.pdf)
- [Apple T2 安全芯片概览](Apple_T2_Security_Chip_Overview.pdf)
- [APFS 文件系统参考手册](Apple-File-System-Reference.pdf)
- [cprotect.h 的开源代码](https://opensource.apple.com/source/xnu/xnu-1699.32.7/bsd/sys/cprotect.h.auto.html)


---


## 参考文献

- [通过侧信道分析加强对iPhone用户身份验证的暴力破解攻击](https://www.anquanke.com/post/id/237769)
