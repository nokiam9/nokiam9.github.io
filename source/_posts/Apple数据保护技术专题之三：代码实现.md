---
title: Apple数据保护技术专题之三：代码实现
date: 2022-11-13 14:34:30
tags:
---

Apple 起家的产品是个人电脑，操作系统是基于 BSD 的定制化Linux，使用的是 MFS 文件系统。

1985年，Apple 发布了 HFS（Hierarchical File System）作为专用的文件系统，后续改进推出了 HFS+（扩展日志式）。HFS 的设计理念源自软盘和磁盘设备时代，为此饱受开发者吐槽，甚至被 Linus 痛斥为有史以来最烂的文件系统，突出的问题有以下几个方面：

- 大小写不敏感（最新版本已支持大小写敏感，但默认配置仍未不敏感）
- 不支持对数据内容进行 checksum 校验
- timestamp 只支持到秒级
- 不支持并发访问，不支持快照，不支持 sparse file
- 使用 big-endian 进行存储

2007年，乔布斯发布了第一代 iPhone，其搭载的操作系统称为 iOS，而个人电脑的操作系统被称为 MacOS。由于 iOS 其实是 MacOS 的一个定制化版本，HFS 只能提供类似 FDE 的全盘加密技术，无法在底层提供文件级别的数据加密能力，许多新功能只能基于上层文件的数据结构，以软件补丁的形式强行实现，开发效率和安全性存在严重隐患。

> 对比 Android 系统，早期采用的**FDE（Full-Disk Encryption）**模式也是一种基于 Volume 的全盘加密技术；从 Android 7 开始，推出 **FBE（File-based Encryption）**模式可以支持文件级别的数据加密，但技术可靠性一直问题多多，直到 Android 13 才基本成熟并取消 FDE 模式的支持，这也从另一个角度说明数据保护技术的复杂性。

2014年，在 Giampaolo 的带领下，Apple启动了APFS（Apple File System）项目研发，并于2017年首次发布，短板终于修补上了！APFS号称是针对SSD等新型设备专门设计（也兼容传统硬盘），提供Copy-on-Write、系统快照、动态分区调整、稀疏大文件支持等功能，但与业界最优秀的ZFS相比并无优势，唯一值得表扬的是，结合Apple封闭硬件体系的优势，执法机构再也没法提取iPhone 的个人数据了。

需要注意的是，以下研究分析大多是基于 iOS 5 之前的版本，随着 iPhone 数据保护技术的多次演进，许多代码已经无法运行，但其技术原理是一脉相承的，仍然具备足够的参考价值。

## 一、文件系统的元数据 - metadata

所有 Linux 的文件系统都是 superblock , inode 和 block 的组合。

- superblock：记录此 filesystem 的整体信息,包括inode/block的总量、使用量、剩余量, 以及文件系统的格式与相关信息等
- inode table：记录文件的权限与属性,一个文件占用一个inode,同时记录此文件的数据所在的 block 号码
- data block：实际记录文件的内容,若文件太大时,会占用多个 block

### 1. inode 的数据结构

Apple 公开了操作系统的部分源代码，可以发现 HFS 仅仅是将 inode 改名为 cnode，但增加了一个 `c_cpentry` 字段用于数据保护。
请参见[https://opensource.apple.com/source/xnu/xnu-1699.32.7/bsd/hfs/hfs_cnode.h](https://opensource.apple.com/source/xnu/xnu-1699.32.7/bsd/hfs/hfs_cnode.h)

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

### 2. cprotect 的数据结构

进一步分析 `cprotect` 的数据结构，包含了文件独有密钥 `cp_persistent_key`（密文）和类标记 `cp_pclass`，并且有持久化存储和运行态的两种格式。
请参见[https://opensource.apple.com/source/xnu/xnu-1699.32.7/bsd/sys/cprotect.h](https://opensource.apple.com/source/xnu/xnu-1699.32.7/bsd/sys/cprotect.h)

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

## 二、核心密钥提取

Github 上有一个取证软件包可以读取早期的 iOS 系统数据。请参见[https://github.com/nabla-c0d3/iphone-dataprotection](https://github.com/nabla-c0d3/iphone-dataprotection)。
基本原理是通过加载修改后的 IPSW 固件，将设备强制进入 DFU 模式，然后通过 SSH 命令行连接到恢复内存盘 (Restore Ramdisk)，从而直接获取系统控制权。
这些工具只能适配 iPhone 4 （iOS 5）之前的版本，随后苹果公司升级软件就无法越狱了。

其中有一段 Python 代码用于读取`Effaceable Storge` 的三个核心密钥，文件路径是 `python_scripts/keystore/effaceable.py`

``` py
Dkey = 0x446B6579
EMF = 0x454D4621
BAG1 = 0x42414731
DONE = 0x444f4e45   #locker sentinel

# Effaceable Stroge 的数据结构
# MAGIC (kL) | LEN (2bytes) | TAG (4) | DATA (LEN)
Locker = Struct("Locker",
                String("magic",2),
                ULInt16("length"),
                Union("tag",
                      ULInt32("int"),
                      String("tag",4))
                ,
                String("data", lambda ctx: ctx["length"])
                )

Lockers = RepeatUntil(lambda obj, ctx: obj.tag.int == DONE, Locker)

class EffaceableLockers(object):
    def __init__(self, data):
        self.lockers = {}
        for l in Lockers.parse(data):
            tag = l.tag.int & ~0x80000000
            tag = struct.pack("<L", tag)[::-1]
            self.lockers[tag] = l.data
    
    # 获取 DKey，使用 AESUnwrap 算法解包
    # 注意！ Python 的 AESUnwrap 函数的入参顺序与 RFC 3394 规范正好相反
    def get_DKey(self, k835):
        if self.lockers.has_key("Dkey"):        
            return AESUnwrap(k835, self.lockers["Dkey"])

    # 获取 EMF，很明显兼容了 LwVM，使用 AESdecryptCBC 算法解密
    def get_EMF(self, k89b):
        if self.lockers.has_key("LwVM"):
            lwvm = AESdecryptCBC(self.lockers["LwVM"], k89b)
            return lwvm[-32:]
        elif self.lockers.has_key("EMF!"):
            return AESdecryptCBC(self.lockers["EMF!"][4:], k89b)
```

下图是 Effaceable Stroge 的数据示例，但是由于版本不同，与上面的 Python 代码并不一致。
浅蓝色是 length ，红色是 tag 标记，注意HFS 是大端字节顺序。
第一个标记：0x42414731 = `BAG1`，长度 0x0034 = 52 个字节
第二个标记：0xC46B6579 = `key`，长度 0x0028 = 40 个字节
第三个标记：0xC54D4621 = `EMF!`，长度 0x0024 = 36 个字节
第四个标记：0x444F4E45 = `DONE`，长度 0x0000，这就是结束了！

![plog](plog.png)

## 三、设备密钥包 - Device Bag

系统密钥包是一个加密的 `plist` 格式的二进制文件，存储了所有类密钥的数据，Apple 系统的数据解密实现方式见下图。

![Daemon](daemon.png)

基于上面的取证工具包，找到了一个读取设备密钥包的小工具。
请参见[https://github.com/russtone/systembag.kb](https://github.com/russtone/systembag.kb) 。

### 1. Device Bag 的文件级解密

系统密钥包的默认存储路径是 `/private/var/keybags/systembag.kb`，如果是U盘引导启动，可能位于`/mnt/keybags/systembag.kb`。
参考其操作步骤，Device Bag 文件解封的处理流程是：

1. 安全隔区从`Effaceable Storage` 区域提取 `BAG1` ，以获得 key 和 iv 初始向量
2. 系统进程`MKBPayload`读取 `systembag.kb` 文件内容并进行解密，获得所有类密钥的密文` Class Key! `

![二进制文件参考](keybag.png)

``` shell
$ get_bag1
iv = e859f45ec0a3ab208ec61477b74e92f0
key = 71ebb0dd387647d7b1c4d10161f5f0b622937867ffe437e41a02ccaacfe8ffb2

$ ./decrypt_systembag.py -k 71ebb0dd387647d7b1c4d10161f5f0b622937867ffe437e41a02ccaacfe8ffb2 -i e859f45ec0a3ab208ec61477b74e92f0 -o example/keybag example/systembag.kb

$ ./parse_keybag.py example/keybag
HEADER
  VERS = 4      // 2- iOS 4.3 ，3 - iOS 5
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
    WRAP = 3    // 3 - UID & Passcode Key 保护
    KTYP = 0
    WPKY = 150dd562e3c6a441a879e154617d758af77553121c2b70114e32f6ad87a5819b375c724adee094ee // 该类密钥的密文
    UUID = 9d4e1c3567cc41058b3d2ee381aaa48b
  1:
    CLAS = 2    // NSFileProtectionCompleteUnlessOpen 类
    WRAP = 3
    KTYP = 1
    WPKY = 0df35185f13495d49596531f38d7114e77134a91c16915a14f2531241a78afc0ae4deeaefd2d5933 // 类密钥，就是静态私钥！
    PBKY = 0252ce8f8acc7068e4ca64cab9227035460ed5cef0661818b382e88609b1a908       // 额外存储了静态公钥！
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

请注意：

- 缺少 Class 4 的类密钥，因为 DKey 的存储路径已经转移到 Effaceable Storage
- 前面 5 个数据项是 Class 类密钥，后面 6 个数据项是 KeyChain 钥匙串的密钥

``` py
PROTECTION_CLASSES={
    # 文件保护类型的定义
    1:"NSFileProtectionComplete",
    2:"NSFileProtectionCompleteUnlessOpen",
    3:"NSFileProtectionCompleteUntilFirstUserAuthentication",
    4:"NSFileProtectionNone",       # 已废弃，DKey 的存储位置改为 Effaceable Storage
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

### 2. Class Key 的解封

![核心流程](unlock2.png)
对于这些 `Class Key!`, 有3种解密方式：

- WRAP = 1 ：只有 Device Key 保护，通过 AES_Decrypt 算法解密
- WRAP = 2 ：只有 Passcode Key 保护，通过 AES_Unwrap 算法解封
- WRAP = 3 ：模式1 + 模式2，首先使用 Passcode Key 解封，再使用 Device Key 解密

更加具体的算法描述，请参见下图。
![unlock](unlock.png)

1. 用户（开机时）必须手工输入 passcode
2. 安全隔区提取 Salt 盐值和 Iter 迭代次数，再次计算 Passcode Key
    `Passcode Key = PBKDF2(passcode, salt, iter, outputLength=32)`
3. 安全隔区使用 Passcode Key，**逐一**解封各个类密钥
    `Class key! = AES_UNWRAP(Passcode Key, Class Key!!)`
4. 类密钥解封完成后，与`systembag.kb`文件头部的 `HMCK`进行校验
    如果失败，可能是 passcode 输入错误，或者是硬件被更换。
5. 安全隔区使用基于 UID 的 Key 0x835 对类密钥进行解密
    `Class key = AES_DECRYPT(Key 0x835, Class Key!)`
6. 现在类密钥已经出现在内存中，可以用于解封文件 metadata 中的`per-file key`

## 四、遗留问题

1. systembag.kb 文件头部包含的 Salt 和 HMCK 字段，是否已经转移到安全隔区的第二代存储组件了呢？

- 标头（Header）:
  - VRES：版本号，例如 3 = iOS 5
  - TYPE：钥匙包类型 ，0 = system, 1 = backup, 3 = escrow, 2 = iCloud Backup
  - UUID：Apple 规定的应用级 UID
  - HMCK：可选，用于数据校验，如果密钥包已签名
  - WRAP：包裹类型，1 = UID保护， 2 = PBKDF2保护
  - 其它字段：可能有 ITER（随机盐），ITER（迭代次数）等

- 类密钥列表（List of class keys）:
  - UUID：Apple 规定的应用级 UID
  - CLAS：文件保护类型（A-D），或者钥匙串保护类型
  - WRAP：包裹类型，1 = 仅UID派生，2 = passcode key，3 = 1&2
  - WPKY：类密钥包裹后的密文
  - 其它字段：可能有非对称的公钥等

---

## 附录：有意思的一些企业信息

1. Sogeti 是凯捷咨询集团（Capgemini）旗下从事本地化技术服务的子公司，1967年在法国创建。该集团还包括 CAP 、GEMINI 和安永咨询等公司，是欧洲最大的IT外包服务商。
  [iPhone数据保护的深度分析 - iPhone Data Protection in Depth](iPhone_Data_Protection_in_Depth.pdf)
2. NCC Group 成立于1999年6月，当时美国国家计算中心（the National Computing Centre）将其商业部门出售给其现有的管理团队，是一个专业的安全审计机构。
  [iOS加密技术高级分析 -  A (not-so-quick) Primer on iOS Encryption](2016-BSidesROC-iOSCrypto.pdf)
3. ElcomSoft Co. Ltd.公司于1990年在美国成立，是国际领先的数字取证工具开发公司。
  [iOS取证技术 - iOS Forensics: Overcoming iPhone Data Protection](OWASP_BeNeLux_Day_2011_-_A._Belenko_-_Overcoming_iOS_Data_Protection.pdf)
  [iOS 数据保护的演变](0721C6_Andrey.Belenko_Evolution.of.iOS.Data.Protection.pdf)
4. 高田科技是一个台湾的科技企业，文档都是中文的，不过是繁体字。
  [iOS取证技术的系列文档](https://www.kaotenforensic.com/ios/)
5. [ibas](https://www.ibas.com/fi) 是一个芬兰的科技公司，主要业务是数据恢复和信息技术取证。  
  [iPhone裸闪存数据恢复的视频演讲 - iPhone raw NAND recovery and forensics](https://www.youtube.com/watch?v=5Es3wRSe3kY)
6. 这是一篇比较完整的技术论文，Peter Teufl 等作者来自于奥地利格拉茨技术大学。
  [iOS加密系统 - iOS Encryption Systems](iOS_Encryption_Systems.pdf)
7. 盘古石，是北京奇安信科技有限公司旗下专注于电子数据取证技术研发的团队。
  [https://qz.qianxin.com/](https://qz.qianxin.com/news-3-1.html)

---

## 参考文献

- [Linux文件系统简介](https://www.cnblogs.com/xumenger/p/4491425.html)
- [ATS-Key-Wrap 算法的 RFC 3394 规范](https://rfc2cn.com/rfc3394.html)
- [解密iPhone的固件](https://blog.csdn.net/XiNGRZ/article/details/5332915)
- [通过侧信道分析加强对iPhone用户身份验证的暴力破解攻击](https://www.anquanke.com/post/id/237769)

### Apple官方文档

- [Apple 平台安全白皮书 - 2022年英文版](apple-platform-security-guide.pdf)
- [Apple 平台安全白皮书 - 2021年中文版](apple平台安全白皮书-2021中文版.pdf)
- [Apple T2 安全芯片概览](Apple_T2_Security_Chip_Overview.pdf)
- [APFS 文件系统参考手册](Apple-File-System-Reference.pdf)
