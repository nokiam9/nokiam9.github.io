---
title: Apple File System的软件加密分析
date: 2024-02-11 18:44:09
tags:
---

## 一、概述

AFPS（Apple File System）是 Apple 平台上使用的默认文件格式。APFS 继承自 HFS+，因此其设计的某些方面有意遵循 HFS+ 以便于历史数据的迁移。

Apple File System 支持对容器、卷和文件使用的数据结构进行加密。当一个卷被加密时，它的文件系统树和该卷中的文件内容都被加密。根据设备的能力，苹果文件系统使用硬件或软件加密：

- 硬件加密：用于支持硬件加密的设备的内部存储，包括macOS(带T2安全芯片)和iOS设备。
    当使用硬件加密时，只有内核（安全隔区）可以与内部存储交互。
    既支持 FDE 单密钥模式，也支持 FBE 多密钥模式。
- 软件加密：用于外部存储（U盘或外接硬盘），以及不支持硬件加密的设备（Intel CPU的Macbook）上的内部存储。
    当使用软件加密时，**仅支持 FDE 单密钥模式**。

大多数应用程序可以使用 Apple 提供的高级接口与文件系统交互，无需自行处理加密和解密，但为了支持跨操作系统的应用（如磁盘备份恢复、Linux系统读取APFS磁盘数据），Apple 提供了[Apple File System Reference](Apple-File-System-Reference.pdf)，公开了技术实现细节，开发者可以自行实现加密和解密处理。

## 二、总体架构

APFS 在概念上分为两层，容器层（Container Layer）和文件系统层（Filesystem Layer）。

所有数据结构采用**小端顺序**存储在磁盘，但设计思路有一些不同。容器对象以 block 为单位，并且包含填充字段使得数据长度是64的倍数，以避免内存访问的性能损失；而文件系统对象以 byte 为单位，并且尽量最小化所使用的空间。

### 1. 容器层 - Container Layer

一个 APFS 分区有一个单独的容器，容器可以包含多个 volume（也称为 filesystem），每个卷都包含一个目录结构，用于管理 file 和 folder。

![L0](level0.png)

- Superblock：一个容器有多个超级块的副本，这些副本保存了容器在过去时间点的状态。
    Block 0 通常是最新的副本，用于在挂载过程中查找检查点。
- Checkpoint area：建立崩溃保护机制。在每个事务结束时，将内存状态写入磁盘并存储超级块的副本。
- Space manager area：跟踪容器内的可用空间，并用于分配和释放存储对象和文件数据的块

### 2. 文件系统层 - Filesystem Layer

文件系统层负责存储文件结构信息，如目录结构、文件元数据和文件内容。
文件系统对象由若干条记录组成，每条记录都是 B-tree 的一个键值对。例如，一个典型的 directory 对象包含由一个 inode 记录、几个目录入口记录和一个扩展属性记录。

![B-Tree](btree.png)

Key 和 Value 从 B-tree 存储区域的首端和尾端开始分别存储，两者之间是共享的自由空间。
Key 和 Value 的位置以 offset 的形式存储，这比存储完整位置使用更少的磁盘空间。

### 3. 密钥包 - Keybag

用于访问文件数据的密钥以 Warpped 状态存储在磁盘上，通过一系列键展开操作访问这些键。

- VEK（Volume Encryption Key，卷加密密钥）：用于访问卷上加密内容的默认密钥，就是那个**单密钥**
- KEK（Key Encryption Key，密钥加密密钥）：用于打开 VEK 的包裹密钥

![L1](keybag.png)

为了不同用途，密钥包中可能存在多个 KEK 的副本，包括：

- User Password：用户输入的登录密码
- Personal recovery key：个人备份密钥，该密钥在驱动器格式化时生成，并由用户纸质保存
- Institutional recovery key：机构恢复密钥
- iCloud recovery key：客户与Apple技术支持配合使用

### 4. inode - 索引信息

File 对象可能有如下类型的记录：

- APFS_TYPE_INODE：必须的，索引信息
- APFS_TYPE_CRYPTO_STATE：文件级加密的 per-file 加密状态
- APFS_TYPE_DSTREAM_ID：data stream
- APFS_TYPE_EXTENT：data stream 的物理信息
- APFS_TYPE_FILE_EXTENT：扩展属性，包含密钥信息
- APFS_TYPE_SIBLING_LINK：从 inode 节点到硬链接的映射
- APFS_TYPE_XATTR：文件名

Directory 对象可能有如下类型的记录：

- APFS_TYPE_INODE：必须的
- APFS_TYPE_CRYPTO_STATE
- APFS_TYPE_DIR_REC
- APFS_TYPE_DIR_STATS
- APFS_TYPE_XATTR

Symbolic Links 对象可能有如下类型的记录：

- APFS_TYPE_INODE：必须的
- APFS_TYPE_XATTR：必须的

## 三、数据结构

### 1. Container Superblock

```c
struct nx_superblock { 
    obj_phys_t nx_o; 
    uint32_t nx_magic; 
    uint32_t nx_block_size; 
    uint64_t nx_block_count; 
    uint64_t nx_features; 
    uint64_t nx_readonly_compatible_features; 
    uint64_t nx_incompatible_features; 
    uuid_t nx_uuid;                 // 容器 UUID
    oid_t nx_next_oid; 
    xid_t nx_next_xid; 
    uint32_t nx_xp_desc_blocks; 
    uint32_t nx_xp_data_blocks; 
    paddr_t nx_xp_desc_base; 
    paddr_t nx_xp_data_base; 
    uint32_t nx_xp_desc_next; 
    uint32_t nx_xp_data_next; 
    uint32_t nx_xp_desc_index; 
    uint32_t nx_xp_desc_len; 
    uint32_t nx_xp_data_index; 
    uint32_t nx_xp_data_len; 
    oid_t nx_spaceman_oid; 
    oid_t nx_omap_oid;
    oid_t nx_reaper_oid; 
    uint32_t nx_test_type; 
    uint32_t nx_max_file_systems; 
    oid_t nx_fs_oid[NX_MAX_FILE_SYSTEMS];   // volume列表
    uint64_t nx_counters[NX_NUM_COUNTERS]; 
    prange_t nx_blocked_out_prange; 
    oid_t nx_evict_mapping_tree_oid; 
    uint64_t nx_flags; 
    paddr_t nx_efi_jumpstart; 
    uuid_t nx_fusion_uuid; 
    prange_t nx_keylocker;      // 容器密钥包的物理位置
    uint64_t nx_ephemeral_info[NX_EPH_INFO_COUNT]; 
    oid_t nx_test_oid; 
    oid_t nx_fusion_mt_oid; 
    oid_t nx_fusion_wbc_oid; 
    prange_t nx_fusion_wbc; 
    uint64_t nx_newest_mounted_version; 
    prange_t nx_mkb_locker;     // Wrapped media key
};
```

### 2. Volume Superblock

```c
struct apfs_superblock { 
    obj_phys_t  apfs_o;
    uint32_t apfs_magic;        // ='BSPA'
    uint32_t apfs_fs_index;     // 容器超级快.nx_fs_oid数组的序号
    uint64_t apfs_features;
    uint64_t apfs_readonly_compatible_features;
    uint64_t apfs_incompatible_features;
    uint64_t apfs_unmount_time;
    uint64_t apfs_fs_reserve_block_count;
    uint64_t apfs_fs_quota_block_count;
    uint64_t apfs_fs_alloc_count;
    wrapped_meta_crypto_state_t apfs_meta_crypto;
    uint32_t apfs_root_tree_type;
    uint32_t apfs_extentref_tree_type;
    uint32_t apfs_snap_meta_tree_type;
    oid_t apfs_omap_oid;                // The physical object identifier of the volumeʼs object map
    oid_t apfs_root_tree_oid;           // B树的入口
    oid_t apfs_extentref_tree_oid;
    oid_t apfs_snap_meta_tree_oid;
    xid_t apfs_revert_to_xid;
    oid_t apfs_revert_to_sblock_oid;
    uint64_t apfs_next_obj_id;
    uint64_t apfs_num_files;
    uint64_t apfs_num_directories;
    uint64_t apfs_num_symlinks;
    uint64_t apfs_num_other_fsobjects;
    uint64_t apfs_num_snapshots;
    uint64_t apfs_total_blocks_alloced;
    uint64_t apfs_total_blocks_freed;
    uuid_t apfs_vol_uuid;               // Volume UUID
    uint64_t apfs_last_mod_time;
    uint64_t apfs_fs_flags;             // Flag定义，必须是 APFS_FS_ONEKEY
    apfs_modified_by_t apfs_formatted_by;
    apfs_modified_by_t apfs_modified_by[APFS_MAX_HIST];
    uint8_t apfs_volname[APFS_VOLNAME_LEN];
    uint32_t apfs_next_doc_id;
    uint16_t apfs_role;                 // volume角色，定义见下
    uint16_t reserved;
    xid_t apfs_root_to_xid;
    oid_t apfs_er_state_oid;
    uint64_t apfs_cloneinfo_id_epoch;
    uint64_t apfs_cloneinfo_xid;
    oid_t apfs_snap_meta_ext_oid;
    uuid_t apfs_volume_group_id;
    oid_t apfs_integrity_meta_oid;
    oid_t apfs_fext_tree_oid;
    uint32_t apfs_fext_tree_type;
    uint32_t reserved_type;
    oid_t reserved_oid;
};

// Define Volume Roles
#define APFS_VOL_ROLE_NONE 0x0000

#define APFS_VOL_ROLE_SYSTEM 0x0001
#define APFS_VOL_ROLE_USER 0x0002
#define APFS_VOL_ROLE_RECOVERY 0x0004
#define APFS_VOL_ROLE_VM 0x0008
#define APFS_VOL_ROLE_PREBOOT 0x0010
#define APFS_VOL_ROLE_INSTALLER 0x0020

/* macOS 10.15、iOS 13 重新定义了 Volume 角色 */
#define APFS_VOL_ROLE_DATA (1 << APFS_VOLUME_ENUM_SHIFT)
#define APFS_VOL_ROLE_BASEBAND (2 << APFS_VOLUME_ENUM_SHIFT)
#define APFS_VOL_ROLE_UPDATE (3 << APFS_VOLUME_ENUM_SHIFT)
#define APFS_VOL_ROLE_XART  (4 << APFS_VOLUME_ENUM_SHIFT)
#define APFS_VOL_ROLE_HARDWARE (5 << APFS_VOLUME_ENUM_SHIFT)
#define APFS_VOL_ROLE_BACKUP (6 << APFS_VOLUME_ENUM_SHIFT)
#define APFS_VOL_ROLE_RESERVED_7 (7 << APFS_VOLUME_ENUM_SHIFT)
#define APFS_VOL_ROLE_RESERVED_8 (8 << APFS_VOLUME_ENUM_SHIFT)
#define APFS_VOL_ROLE_ENTERPRISE (9 << APFS_VOLUME_ENUM_SHIFT)
#define APFS_VOL_ROLE_RESERVED_10 (10 << APFS_VOLUME_ENUM_SHIFT)
#define APFS_VOL_ROLE_PRELOGIN 11 << APFS_VOLUME_ENUM_SHIFT)

#define APFS_VOLUME_ENUM_SHIFT 6
```

### 3. Keybag

```c
// A keybag
struct kb_locker { 
    uint16_t kl_version; 
    uint16_t kl_nkeys;              // 包含了几个entry
    uint32_t kl_nbytes;
    uint8_t padding[8]; 
    keybag_entry_t kl_entries[];    // 密钥入口！长度不定，结构见下
};

// Entry in a Keybag
struct keybag_entry { 
    uuid_t ke_uuid;                 // 如果是容器，存储Volume UUID；如果是Volume，存储User UUID
    uint16_t ke_tag;                // 数据标记！！！
    uint16_t ke_keylen;             // 
    uint8_t padding[4]; 
    uint8_t ke_keydata[];           // 内容数据块，长度是ke_keylen
};
typedef struct keybag_entry keybag_entry_t;

// Define of ke_tag
enum {
    KB_TAG_UNKNOWN = 0, 
    KB_TAG_RESERVED_1 = 1, 
    KB_TAG_VOLUME_KEY = 2,              // 存储wrapped VEK，仅用于容器密钥包
    KB_TAG_VOLUME_UNLOCK_RECORDS = 3,   // 如果是容器密钥包，存储volume keybag的地址；如果是卷宗密钥包，存储wrapped KEK
    KB_TAG_VOLUME_PASSPHRASE_HINT = 4,  // 存储user password的提示语（明文），仅适用于MacOS！
    KB_TAG_WRAPPING_M_KEY = 5,          // 存储media key的包裹密钥，适用于iOS！
    KB_TAG_VOLUME_M_KEY = 6,            // 存储本volume的media key，仅适用于iOS！
    KB_TAG_RESERVED_F8 = 0xF8 
};
```

### 4. Directory Objects

使用 APFS 格式运行的设备可能支持文件克隆 (使用写入时拷贝技术的零损耗拷贝)。如果文件被克隆，克隆的每一半都会得到一个新的密钥以接受传入的数据写入，这样新数据会使用新密钥写入介质。久而久之，文件可能会由不同的范围（或片段）组成，每个映射到不同的密钥。

```c
/* inode 数据结构 */
struct j_inode_key { 
    j_key_t hdr;
} __attribute__((packed));

struct j_inode_val { 
    uint64_t parent_id;
    uint64_t private_id;
    uint64_t create_time;
    uint64_t mod_time;
    uint64_t change_time;
    uint64_t access_time;
    uint64_t internal_flags;
    union { 
        int32_t nchildren;
        int32_t nlink;
    };
    cp_key_class_t default_protection_class;
    uint32_t write_generation_counter;
    uint32_t bsd_flags;
    uid_t owner;
    gid_t group;
    mode_t mode;
    uint16_t pad1;
    uint64_t uncompressed_size;
    uint8_t xfields[];
} __attribute__((packed));
typedef struct j_inode_val j_inode_val_t;

/* APFS_TYPE_FILE_EXTENT 数据结构 */
struct j_file_extent_key { 
    j_key_t hdr;                        // header，= APFS_TYPE_FILE_EXTENT
    uint64_t logical_addr;
} __attribute__((packed));
typedef struct j_file_extent_key j_file_extent_key_t;

struct j_file_extent_val { 
    uint64_t len_and_flags; 
    uint64_t phys_block_num;            // Tweak value!
    uint64_t crypto_id;                 // Tweak Key!
} __attribute__((packed));

/* APFS_TYPE_CRYPTO_STATE 数据结构 */
struct j_crypto_key { 
    j_key_t hdr;
} __attribute__((packed));

struct j_crypto_val {
    uint32_t refcnt; 
    wrapped_crypto_state_t state;
} __attribute__((aligned(4),packed)); 

struct wrapped_meta_crypto_state {
    uint16_t major_version;
    uint16_t minor_version;
    crypto_flags_t cpflags;
    cp_key_class_t persistent_class;
    cp_key_os_version_t key_os_version;
    cp_key_revision_t key_revision;
    uint16_t  unused;
} __attribute__((aligned(2), packed));

struct wrapped_crypto_state {
    uint16_t major_version;
    uint16_t minor_version;
    crypto_flags_t cpflags;
    cp_key_class_t persistent_class;
    cp_key_os_version_t key_os_version;
    cp_key_revision_t key_revision;
    uint16_t key_len；
    uint8_t persistent_key[0];          // The wrapped key data
} __attribute__((aligned(2), packed));
```

## 四、核心处理流程

### 1. 如何获取 VEK

1. 查找容器密钥包的物理位置，位于`nx_superblock.nx_keylocker`字段
2. Container Keybag 解封：AES_UNWRAP(`nx_superblock.nx_uuid`, container keybag的密文)
3. 查找容器密钥包`kb_locker`的一个记录：
   `ke_uuid == volume UUID && ke_tag == KB_TAG_VOLUME_KEY`，
   此记录的`ke_keydata`字段就是这个 Volume 的 `wrapped VEK`
4. 查找容器密钥包`kb_locker`的一个记录：
    `ke_uuid == volume UUID && ke_tag == KB_TAG_VOLUME_UNLOCK_RECORDS`，
    此记录的`ke_keydata`字段就是 Volume keybag 的物理位置
5. Volume Keybag 解封：AES_UNWRAP(volume_uuid, volume keybag的密文)
6. 查找卷宗密钥包`kb_locker`的一个记录：
   `ke_uuid == User Open Directory UUID && ke_tag == KB_TAG_VOLUME_UNLOCK_RECORDS`，
   此记录的`ke_keydata`字段就是这个 Volume 的 `warpped KEK`
7. KEK = AES_UNWRAP(User password,`warpped KEK`)
   VEK = AES_UNWRAP(KEK, `wrapped VEK`)

### 2. 如何解密文件

1. 使用 VEK 作为AES-XTS 主密钥，解密存储卷的根文件系统树的块，通过`apfs_superblock.apfs_root_tree_oid`字段访问文件系统树
2. 查找加密文件的文件范围记录`APFS_TYPE_FILE_EXTENT` 
3. 查找加密状态记录`APFS_TYPE_CRYPTO_STATE`，其标识符等于`j_file_extent_val.crypto_id`
4. 使用 VEK 作为主密钥，`Crypto_id`的值作为 Tweak key，执行 AES-XTS 解密相应的数据块

---

## 参考文献

- [RFC 3394 - AES密钥包裹算法](https://www.rfc-editor.org/rfc/rfc3394)