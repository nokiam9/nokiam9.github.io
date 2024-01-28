---
title: Apple Data Protection的深度分析
date: 2024-01-28 20:05:45
tags:
---

花了好长时间，才明白Apple公司数据保护（Data Protection）的核心。

根据Apple安全白皮书的官方文档，数据保护实际上有两条路径，一是面向智能终端的 iOS 发展来的 Data Protection，二是面向个人电脑的 MacOS 发展来的数据保险箱，由于依赖Intel CPU，数据安全就是各种打补丁。M1 发布后两条路径开始融合，Secure Encalve 和 AFPS 成为融合的基石。
任何技术都有路径依赖，虽然 HFS+ 被认为是业界最烂的文件系统，但其许多重要的核心功能仍然需要 AFPS 继承下来，当然也为后续发展打下基础。

![更细架构](arch2.png)

- 每个文件创建时会生成一个`Per-file Key`，在文件写入闪存时通过硬件用`AES-XTS`算法加密
- `Per-file Key`存储在文件元数据 Metadata 中，被`Class Key`和`Filesystem Key`加密
- `Class Key`和`Filesystem Key`都被硬件密钥`UID`保护
- `Class Key`还被用户密码`Passcode`保护

实际上，Apple就是在 HFS+ 文件系统的 inode 中增加了一个`cprotect` 字段：

- 如果数据保护级别是 A / B / C，存储以`Class Key`加密的`Per-file Key`
- 如果数据保护级别是 D，存储加密后的`DKey`(Device Key，正好也是D)
- 如果设置为`discarded`，则意味着密码失效，文件就永远无法打开，也是数据被抹去了...

## 一. 安全隔区有那些内置的核心密钥？

![密钥关系图](keys.png)

### 1. UID & GID

- UID 是一个AES 256 位密钥，在 SOC 制造过程中写入一次性的**熔丝**，每个设备唯一且无法更改
- UID 不能被固件或软件读取，只能由处理器的硬件 AES 引擎使用
- Apple 或其任何供应商都不会记录 UID
- UID 与设备上的任何其他标识符无关，包括但不限于 UDID

> `UDID = SHA1(Serial Number + ECID + LOWERCASE (WiFi Address) + LOWERCASE(Bluetooth Address))`

#### 派生的硬件密钥

安全隔区启动时，将从 UID 派生出多个硬件密钥，加密因子是不同的固定盐。
这些派生的硬件密钥都在内存中，无需持久化存储。
在各种业务场景下，使用不同的派生密钥，而不是直接使用 UID，可以有效减少 UID 被泄露的风险。

- Key 0x835 = AES(UID, 01010101010101010101010101010101)；保护`Class key`，也被称为`device key`
- Key 0x836 = AES(UID, 00E5A0E6526FAE66C5C1C6D4F16D6180)
- Key 0x837 = AES(GID, 345A2D6C5050D058780DA431F0710E15)
- Key 0x838 = AES(UID, 8C8318A27D7F030717D2B8FC5514F8E1)
- Key 0x89B = AES(UID, 183e99676bb03c546fa468f51c0cbd49)；保护`EMF key`

### 2. Passcode & Passcode Key = KEK

Passcode Key是用戶輸入的passcode結合系統硬件的加密引擎以及PBKDF2(Password-Based Key Derivation Function)算法生成的。PBKDF2的基本原理是通過一個偽隨機函數，把明文和一個鹽值及加密重複次數作為輸入參數，然後重複進行運算，並最終產生密鑰。重複運算的會使得暴力破解的成本變得很高，而硬件key及鹽值的添加基本上斷絕了通過“彩虹表”攻擊的可能。

`Passcode key = PBKDF2(SHA256, passcode, UID, iter=1, outputLength=32)`
![passcode-key-kdf](passcode-key.png)

> KEK，密钥加密密钥，好像就是Passcode Key，但是可能只限于Mac OS，而非 iOS？

Passcode-derived key (PDK) The encryption key derived from the entangling of the user password with the long-term SKP key and the UID of the Secure Enclave.

### 3. xART key

扩展反重放技术的密钥（eXtended Anti-Replay Technology key），就是第二代安全存储组件的**唯一加密密钥**.
后续苹果为了封堵各种暴力猜测Passcode的方法，在64位设备的Secure Enclave中增加了定时器，针对尝试密码的错误次数，增加尝试的延时，即使断电重启也无法解决。


## 二. 安全隔区访问的`Secure Nonvolatile Storage`是什么？

![基本结构](arch3.jpg)

CPU都是通用的，这些关键密钥都是个性化数据，必须有一个存储设备提供数据持久化！
秘密就藏在 NAND 闪存的1号扇区，Apple 将它作为隐藏的可安全擦除区域(Effaceable Storage)，就是安全隔区的技术架构中的`Secure nonvolatile storage`。
> Effaceable Storage：accesses the underlying storage technology (for example, NAND) to directly address and erase a small number of blocks at a very low level.

当然，Apple官方文档说安全隔区通过专用 I2C 总线连接，但 NAND 闪存不太可能再增加一个物理接口，也许是复用吧。
这个隐藏存储区域的容量是 960 Bytes，存储了3个关键数据。

### 1. EMF key = LwVM = VEK

在首次系统安装时创建的原生随机数 `FileSystem Key`，以 `key 0x89B` 包裹形成密文 `EMF Key`，并持久化存储在 NAND 的可擦除分区。
EMF Key 负责数据分区（Data Partition）的加密，也称为卷宗加密密钥 VEK（Volume Encryptition Key）
在iOS 5之后也被称为 `LwVM`（Lightweight Volume Manager）

- 功能描述：**数据分区**（data partition）的加密密钥，
- 构造方式：原生随机数，**在首次安装操作系统或被用户擦除设备时创建（ToDO：？）**
- 构造方式：`EMF Key = AES(FileSystem Key, key 0x89B)`
- 储存位置：NAND闪存的可擦除区域

> EMF key used for filesystem key is derived from key 0x89B - can be read by reading the LWVM locker(0x4C77564d) in EffaceableStorage - see iphone data protection project for how get this.

### 2. Dkey

设备密钥，Device key

- 功能描述：就是`NSProtectionNone` 类密钥的封装密钥，主要用于远程数据擦除
- 构造方式：原生随机数
- 加密方式：decrypt(Dkey, key 0x835)
- 储存位置：NAND闪存的可擦除区域

> 闪存Flash的特点是wear-leveling，删除数据很困难，但加密了就容易多了，直接删除密钥就OK

### 3. BAG1: system bag

- 功能描述：系统密钥包（`/private/var/keybags/systembag.kb`）的文件级密钥，并包含一个初始向量IV
- 构造方式：原生随机数
- 加密方式：**不加密**。系统密钥包实际是一个数据文件，保存的Class Key还有一层加密
- 储存位置：NAND闪存的可擦除区域

系统密钥包有效负载密钥 （+初始化向量）。未加密地存储在可擦除区域。

### 4. NAND Key

负责加密GPT分区表和系统分区表，也被称为媒体密钥（media key）

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

---


salads

- `BAG1`: System Keybag payload key and IV
- `Dkey`: NSProtectionNone class master key
- `EMF!`: Filesystem encryption key


### ios 4

- Only User partition is encrypted
- Available protection classes:
  - NSProtectionNone
  - NSProtectionComplete
- When no protection class set, EMF key is used – Filesystem metadata and unprotected files
  - Transparent encryption and decryption (same as pre-iOS 4)
- When protection class is set, per-file random key is used
  - File key protected with master key is stored in extended attribute com.apple.system.cprotect

### ios Storage

• New partition scheme
– “LwVM” – Lightweight Volume Manager
• Any partition can be encrypted • New protection classes
– NSFileProtectionCompleteUntilFirstUserAuthentication – NSFileProtectionCompleteUnlessOpen
• IV for file encryption is computed differently



### 2. EMF key

密钥：数据分区加密密钥。也称为“媒体密钥”。通过密钥0x89B加密存储

### Passcode Key

Passcode Key是用戶輸入的passcode結合系統硬件的加密引擎以及PBKDF2(Password-Based Key Derivation Function)算法生成的。PBKDF2的基本原理是通過一個偽隨機函數，把明文和一個鹽值及加密重複次數作為輸入參數，然後重複進行運算，並最終產生密鑰。重複運算的會使得暴力破解的成本變得很高，而硬件key及鹽值的添加基本上斷絕了通過“彩虹表”攻擊的可能。

由於硬件加密引擎的Key無法提取，所以只能在目標的機器上運行暴力破解程序進行破解，假設用戶的密碼設置的足夠複雜的話，那麼破解的周期就會變得非常久。

KDF = Deriving Key from Password

密码密钥：使用 Apple 自定义派生函数 (Tangling) 从用户密码或托管密钥包 BagKey 计算得出。用于从系统/托管密钥包中解开类密钥。解开钥匙包钥匙后立即从内存中删除。



在支持数据保护的 Apple 设备上， 密钥加密密钥 (KEK) 既受系统上软件测量值的保护 （或密封）， 又与只能从安全隔区获得的 UID 绑定。 在搭载 Apple 芯片的 Mac 上， 对 KEK 的保护通过整合有关系统安全性策略的信息进一步得到了加强， 因为 macOS 支持其他平台不支持的关键安全性策略更改 （例如， 停用安全启动或 SIP）。 在搭载 Apple 芯片的 Mac 上， 由于文件保险箱的实施使用数据保护 （C 类）， 此保护涵盖文件保险箱密钥。
On Apple devices that support Data Protection, the key encryption key (KEK) is protected (or sealed) with measurements of the software on the system, as well as being tied to the UID available only from the Secure Enclave.

xART An abbreviation for eXtended Anti-Replay Technology.

![SKP](SKP.png)
![SKP](SKP-C.png)

SKP = 密封密钥保护，也称为操作系统绑定密钥
KEK = key encryption key，密钥加密密钥
VEK = volume encryption key，卷宗加密密钥
xART key = eXtended Anti-Replay Technology key，扩展反重放技术的密钥，就是第二代安全存储组件的**唯一加密密钥**

SMRK = the crypto-hardware-derived System Measurement Root Key (SMRK)，系统测量根密钥，从加密硬件派生
SMDK = the system measurement device key，系统测量设备密钥，

PDK = Passcode-derived key，用户密码派生密钥
？ Key Wrapping Key, 密钥封装密钥。
  随机宗卷密钥由密钥封装密钥加密和封装。
  密钥封装密钥由安全隔区长期储存， 只在安全隔区中可见。 每次用户抹掉设备时， 它都会发生变化。

> 在 Apple A10 及后续型号的 SoC 上， PKA 支持操作系统绑定密钥， 也称为密封密钥保护 (SKP)。
> 这些密钥基于设备 UID 和设备上所运行 sepOS 的哈希值组合生成。
> 哈希值由安全隔区 Boot ROM 提供， 在Apple A13 及后续型号的 SoC 上， 则由安全隔区启动监视器提供。
> 这些密钥还用于在请求特定 Apple 服务时验证 sepOS 版本， 以及用于在系统发生重大更改而未获得用户授权时通过协助阻止访问密钥材料来提高受密码保护数据的安全性。

关键0x835：由内核在引导时计算。仅用于 iOS 3 及更低版本中的钥匙串加密。用作“设备密钥”，用于保护 iOS 4 中的类密钥。

key835 = AES(UID, bytes("01010101010101010101010101010101"))
关键0x89B：由内核在引导时计算。用于加密存储在闪存上的数据分区键。防止直接从 NAND 芯片读取数据分区键。

key89B = AES(UID, bytes("183e99676bb03c546fa468f51c0cbd49"))

DKey： NSProtectionNone class key.用于包装 iOS 4 中数据分区上“始终可访问”文件的文件名。由密钥0x835包装存储

BAG1 密钥：系统密钥包有效负载密钥 （+初始化向量）。未加密地存储在可擦除区域。

密码密钥：使用 Apple 自定义派生函数 （Tangling） 从用户密码或托管密钥包 BagKey 计算得出。用于从系统/托管密钥包中解开类密钥。钥匙包钥匙解开包装后立即从内存中删除。

文件系统密钥（f65dae950e906c42b254cc58fc78eece）：用于加密分区表和系统分区（图中称为“NAND 密钥”）

元数据密钥（92a742ab08c969bf006c9412d3cc79a5）：加密NAND元数据（vfl / ftl上下文和索引页面）

---
The keys used to access file data are stored on disk in a wrapped state. You access these keys through a chain of key-unwrapping operations. The volume encryption key (VEK) is the default key used to access encrypted content on the volume. **The key encryption key (KEK) is used to unwrap the VEK**. The KEK is unwrapped in one of several ways:

- User password. The user enters their password, which is used to unwrap the KEK.
- Personal recovery key. This key is generated when the drive is formatted and is saved by the user on a paper printout. The string on that printout is used to unwrap the KEK.
- Institutional recovery key. This key is enabled by the user in Settings and allows the corresponding corporate master key to unwrap the KEK.
- iCloud recovery key. This key is used by customers working with Apple Support, and isnʼt described in thisdocument.

For example, to access a file given the userʼs password on a volume that uses per-volume encryption, the chain of key unwrapping and data decryption consists of the following high-level operations:

1. Unwrap the KEK using the userʼs password.
2. Unwrap the VEK using the KEK.
3. Decrypt the file-system B-tree using the VEK.
4. Decrypt the file data using the VEK.

The detailed steps are described in Accessing Encrypted Objects below.

> KEK - 密钥加密密钥，似乎就是 Passcode Key；VEK - 卷宗加密密钥，似乎就是 EMF Key
---

### FileSystem Key：文件系统密钥

- 用于加密 `GPT`分区表 和 系统分区表`System Partition`
- 也被称为 `NAND Key`

> file system key The key that encrypts each file’s metadata, including its class key. This is kept in Effaceable Storage to facilitate fast wipe, rather than confidentiality.

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

[https://www.adeleda.com/ios-forensics-en.html]

从iPhone 3GS开始，iDevices包含一个加密芯片，可以对文件系统进行硬件加密。 NAND芯片是按如下方式组织的闪存：

- 块 0：包含 LLB
- 块 1：包含以下加密密钥：
  - EMF：用于加密文件系统
  - Dkey：用于加密保护类“NSFileProtectionNone”的主密钥（大多数文件）
  - BAG1：与密码一起使用，为其他主密钥生成加密密钥（对于邮件等文件...
- 块 16 到 （END-15） 块：包含 HFS+ 文件系统
- 最后 15 个区块：保留给 Apple 用于其他用途。

---

Effaceable lockers
EMF!
• Data partition encryption key, encrypted with key 0x89B
• Format: length (0x20) + AES(key89B, emfkey)
Dkey
• NSProtectionNone class key, wrapped with key 0x835
• Format: AESWRAP(key835, Dkey)
BAG1
• System keybag payload key
• Format : magic (BAG1) + IV + Key
• Read from userland by keybagd to decrypt systembag.kb
• Erased at each passcode change to prevent attacks on previous keybag

---

Data Wipe
Operation
• mobile_obliterator daemon
• Erase DKey by calling MKBDeviceObliterateClassDKey
• Erase EMF key by calling selector 0x14C39 in EffacingMediaFilter service
• Reformat data partition
• Generate new system keybag
• High level of confidence that erased data cannot be recovered

---

### 4. 遗留问题

1. 用户修改passcode后，需要重新封装metadata中所有的per-file key，似乎计算量也不小啊！


### 2. 关于 iOS 的文件保护类型

对于早期的 iOS 系统，Class D 是用户数据文件的默认类型，其类密钥是 DKey。也就是说，尽管每个文件都有 per-file key，但是这些文件专有密钥的包裹密钥都是 DKey，而 DKey 并未将 Passcode 作为加密因子（仅由基于 UID 的 Key 0x835 提供保护），因此其加密强度难以抵抗暴力破解。

从 iOS 5 开始，Apple 决定将 Class C 作为默认类型，即在设备开机时，校验用户输入的 passcode，成功后即可自由访问（即使发生了锁屏事件，仍然可以访问数据文件）；但是如果设备关机，就必须在开机之后重新校验 passcode。

Class A 主要适用于日历、信息、邮件、照片、通讯录和健康数据等个人敏感信息，其特点是一旦锁屏事件发生（延迟10秒），将丢弃内存中的类密钥，结果是无法访问数据文件了；只有用户重新输入 passcode 解锁屏幕后，再重新计算类密钥，才可以继续提供数据文件访问。

Class B 的设计目标是解决远程下载等互联网服务的后台权限问题，方法是采用 ECC 非对称密码系统，代价是文件元数据需要额外存储一个临时公钥（类密钥就是静态私钥）

---


## 六、技术分析

### 1. SKP 密封密钥保护技术

LLB - Low Level Bootloader，底层引导载入程序

在具有两步启动架构的 Mac 电脑上，LLB 包含由 Boot ROM 调用的代码，该代码随后会载入 iBoot，成为安全启动链的一环。

![搭载 Apple 芯片的 Mac 开机时的启动过程步骤](mac-boot.png)

如果攻击者能够意外地更改任何上述测量的固件、 软件或安全性配置组件， 则也会修改储存在硬件寄存器中的测量值。 测量值的修改会导致从加密硬件派生的系统测量根密钥 (SMRK) 派生出不同的值， 从而有效破坏密钥层 级的封章。 这将导致无法访问系统测量设备密钥 (SMDK)， 从而导致无法访问 KEK， 因此也无法访问数据。

但是， 系统在未受到攻击时， 必须容纳合法的软件更新， 这些更新会更改固件测量值和 LocalPolicy 中的 nsih 字段， 以指向新的 macOS 测量值。 在其他尝试整合固件测量值但没有已知真实来源的系统中， 用 户将被要求停用安全性， 更新固件后重新启用安全性， 以便捕获新的测量基线。 这大大增加了攻击者在软件 更新期间篡改固件的风险。 Image4 清单包含所需的所有测量值， 这对系统很有帮助。 正常启动期间如果测 量值匹配， 使用 SMRK 解密 SMDK 的硬件也可以将 SMDK 加密为所建议的将来的 SMRK。 通过指定软 件更新后预期的测量值， 硬件可以加密在当前操作系统中可访问的 SMDK， 以便在将来的操作系统中仍可 访问。 同样地， 当客户在 LocalPolicy 中合法更改其安全性设置时， 必须根据 LLB 下次重新启动时计算的 LocalPolicy 测量值， 将 SMDK 加密为将来的 SMRK。

> Image4 是经 ASN.1（抽象语法标记一）DER 编码的数据结构格式，用于描述 Apple 平台上有关安全启动链对象的信息。

---

EMF Key = file-system master encryption key

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


## 破解Passcode Key的手段

假设把每个文件的加密看作为上了一道锁的话，那么对应的开锁的钥匙就存放在系统密钥包里面，而锁屏密码除了防止用户进入系统桌面之外，更重要的角色就是利用密码对系统密钥包进行额外的加密保护。很多人对锁屏密码理解的一个误区，就是锁屏密码只是物理手段上防止进入手机的一个保护，但实际上，用户在第一次设置锁屏密码的时候，锁屏密码会结合硬件加密引擎生成一个叫做Passcode Key的密钥，通过这个密钥对保存在密钥包中的各个钥匙(Class Key)进行加密保护。锁屏密码不会以其他加密的形式保存在设备上，用户在解锁的时候，会直接用输入的密码生成Passcode Key对密钥包中的Class Key解密，解密失败代表用户密码错误。

从苹果的数据加密和锁屏密码的保护机制来看，直接拆除存储芯片并对其进行文件读写操作是不可能的。

Passcode Key是用户输入的passcode结合系统硬件的加密引擎以及PBKDF2(Password-Based Key Derivation Function)算法生成的。PBKDF2 的基本原理是通过一个伪随机函数，把明文和一个盐值及加密重复次数作为输入参数，然后重复进行运算，并最终产生密钥。重复运算的会使得暴力破解的成本变得很高，而硬件key及盐值的添加基本上断绝了通过“彩虹表”攻击的可能 。

由于硬件加密引擎的Key无法提取，所以 只能在目标的机器上运行暴力破解程序进行破解，假设用户的密码设置的足够复杂的话，那么破解的周期就会变得非常久。

在FBI这个案例中，由于嫌犯可能开启了输错10次密码自动擦除设备的选项，一旦暴力猜测程序连续10次输入错误的密码，设备上的所有内容就会擦除掉。一旦触发数据擦除，苹果会首先对可安全擦除区域进行擦除，物理方式上即使能够恢复大部分加密数据，但是却无法恢复可安全擦除区域中的数据，因为大部分的解密密钥都保存在这个区域中，例如能够解开系统密钥包的二进制数据的BAG1 Key。

后续苹果为了封堵各种暴力猜测Passcode的方法，在64位设备的Secure Enclave中增加了定时器，针对尝试密码的错误次数，增加尝试的延时，即使断电重启也无法解决。

---

## 名词解释

### Effaceable Storage，可擦除区域

可擦除存储器 NAND 存储器中一个用于储存加密密钥的专用区域，可被直接寻址和安全擦除。尽管当攻击者实际占有设备时，可擦除存储器无法提供保护，但其中存储的密钥可用作密钥层级的一部分，用于实现快速擦 除和前向安全性。

### xART - eXtended Anti-Replay Technology，反重放技术

一组为具有反重放功能 (基于物理储存架构) 的安全隔区提供加密且经认证的永久储存区的服务。

### UID - unique ID，唯一ID

UID 是一个 256 位的 AES 密钥，在设备制造过程中刻录在每个处理器上。
这种密钥无法由固件或软件读取，只能由处理器的硬件 AES 引擎使用。 若要获取实际密钥，攻击者必须对处理器的芯片发起极为复杂且代价高昂的物理攻击。 UID 与设备上的任何其他标识符均无关，包括但不限于 UDID。

#### PDK - Passcode-Derived Key，密码派生密钥

用户密码与长期 SKP 密钥和安全隔区的 UID 配合使用，由此派生加密密钥。

#### per-file key，文件独有密钥

数据保护用于在文件系统上加密文件的密钥。文件独有密钥使用类密钥封装，储存在文件的元数据 metadata 中。

#### filesystem key，文件系统密钥

用于加密每个文件的元数据的密钥，包括其类密钥。存储在可擦除存储器中，用于实现快速擦除，并非用于保密目的。

#### Volume Key，卷宗密钥

数据宗卷文件系统中所有文件的元数据都使用随机宗卷密钥（Volume Key）进行加密，该密钥在首次安装操作系统或用户擦除设备时创建。

#### KEK - Key Encryption Key，密钥封装密钥

宗卷密钥密钥由密钥封装密钥（Key Encryption Key）加密和封装，密钥封装密钥由安全隔区长期储存，只在安全隔区中可见。每次用户抹掉设备时，它都会发生变化。在 A9（及后续型号） SoC 上，安全隔区依靠由反重放系统支持的熵来实现可擦除性，以及保护其他资源中的密钥封装密钥。

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