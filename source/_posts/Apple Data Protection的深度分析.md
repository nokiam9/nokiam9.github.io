---
title: Apple Data Protection的深度分析
date: 2024-01-28 20:05:45
tags:
---

花了好长时间，才明白Apple公司数据保护（Data Protection）的核心。

## 一、FDE Vs FBE

根据Apple安全白皮书的官方文档，数据保护实际上有两条路径，一是面向智能终端的 iOS 发展来的 Data Protection，二是面向个人电脑的 MacOS 发展来的数据保险箱，由于依赖Intel CPU，数据安全就是各种打补丁。M1 发布后两条路径开始融合，Secure Encalve 和 AFPS 成为融合的基石。
任何技术都有路径依赖，虽然 HFS+ 被认为是业界最烂的文件系统，但其许多重要的核心功能仍然需要 AFPS 继承下来，当然也为后续发展打下基础。

## 二. Secure Nonvolatile Storage

CPU都是通用的，许多关键密钥都是个人化数据，必须有一个固定的物理空间提供持久化存储，秘密就藏在 NAND 闪存的`Block 1`！

![ES](ES.png)

从iPhone 3GS开始，iPhone 内置的闪存芯片都最前面的16个block和最后的15个block保留给Apple，并将`Block 1`作为 Effaceable Storage（可擦除区域），用于储存加密密钥的专用区域，可被直接寻址和安全擦除，这就是 Secure Nonvolatile Storage（非易失性存储）。
> Apple官方文档说安全隔区通过专用 I2C 总线连接，但 NAND 闪存不太可能再增加一个物理接口，也许是复用吧。

尽管当攻击者实际占有设备时，可擦除存储器无法提供保护，但其中存储的密钥（例如：DKey）可用作密钥层级的一部分，用于实现快速擦除和前向安全性。

## 三、Secure Encalve 中的密钥

![密钥关系图](keys.png)

### 1. UID & GID

UID 是一个AES 256 位密钥，在 SOC 制造过程中写入，每个设备唯一且无法更改。
随机生成的 UID 在制造过程中便被固化到 SoC 中。从 A9 SoC 开始，UID 在制造过程中由安全隔区TRNG 生成，并使用完全在安全隔区中运行的软件进程写入到一次性的**熔丝**熔丝中。 
UID 不能被固件或软件读取，只能由安全隔区的硬件 AES 引擎使用，Apple 或其供应商都不会存储 UID。

安全隔区还长期存储 GID（Group ID，设备组 ID），它是使用特定 SoC 的所有设备共用的 ID（例如，所有使用 Apple A14 SoC 的设备共用同一个 GID）。

UID 与设备上的任何其他标识符无关，包括但不限于 UDID。
UID 和 GID 不可以通过联合测试行动小组 (JTAG) 或其他调试接口使用。

#### 2. 派生的硬件密钥

基于单一的密钥材料（UID 或 GID），在各种业务场景使用不同的派生密钥，而不是直接使用 Root 密钥，可以有效减少核心密钥被泄露的风险，这就是密钥多样化。
Apple Encalve 启动时派生（很可能是一次性计算）出多个子密钥，这些密钥保存在安全隔区的 AES 引擎内部，即使对 sepOS 软件也不可见。
派生密钥的加密因子是不同的 Salt，取值是固定的，例如：

- Key 0x835 = AES_WRAP(UID, 0x'01010101010101010101010101010101')
- Key 0x836 = AES_WRAP(UID, 0x‘00E5A0E6526FAE66C5C1C6D4F16D6180’)
- Key 0x837 = AES_WRAP(GID, 0x'345A2D6C5050D058780DA431F0710E15’)
- Key 0x838 = AES_WRAP(UID, 0x'8C8318A27D7F030717D2B8FC5514F8E1‘)
- Key 0x89B = AES_WRAP(UID, 0x'183e99676bb03c546fa468f51c0cbd49')

其中，`Key 0x89B`的功能类似 FDE，用于**包裹**存储在闪存上的`EMF`密钥，防止被黑客从闪存芯片上直接读取；`Key 0x835`用于保护`DKey`；`Key 0x837`则用于苹果固件的img3文件的解密。

### 3. Passcode Key

用户的锁屏密码 Passcode 的长度有限，不能直接作为密钥使用，必须经过密钥扩展生成标准的 Passcode Key，核心算法是 PBKDF2（Password-Based Key Derivation Function）算法。

`DK = PBKDF2(PseudoRandomFunction, password, salt, iters, dk_len)`
基本原理是通过一个伪随机函数（不限定具体算法，哈希算法和对称密钥算法都可以），把明文和一个随机盐作为输入参数，然后重复（至少1000次）进行运算，并最终产生密钥。大量重复运算使得暴力破解的成本很高，而添加盐值可以有效防御**彩虹表**攻击。

注意！不同型号的 Apple 设备的 KDF 算法存在较大差异。

- iOS 4 的伪随机算法是以 UID 为密钥的 AES 对称加密算法，而且 iOS 4.3 升级了 Keybag，版本 v2 的算法也有变化
![passcode-key-kdf](passcode-key.png)

- Filevault 2 的 KEK 也基于 Passcode Key 包裹，但其采用的 PBKDF2 算法基于 HMAC-SHA256，迭代次数为41000，加密因子不包含 UID！

## 四、Effaceable Storage 中的密钥

![基本结构](arch3.jpg)

### 1. EMF = LwVm

在首次系统安装时创建的原生随机数 `FileSystem Key`，以 `key 0x89B` 包裹形成密文 `EMF Key`，并持久化存储在 NAND 的可擦除分区。
EMF Key 负责数据分区（Data Partition）的加密，也称为卷宗加密密钥 VEK（Volume Encryptition Key）
在iOS 5之后也被称为 `LwVM`（Lightweight Volume Manager）

- 功能描述：**数据分区**（data partition）的加密密钥，
- 构造方式：原生随机数，**在首次安装操作系统或被用户擦除设备时创建（ToDO：？）**
- 构造方式：`EMF Key = AES(FileSystem Key, key 0x89B)`
- 储存位置：NAND闪存的可擦除区域

> EMF key used for filesystem key is derived from key 0x89B - can be read by reading the LWVM locker(0x4C77564d) in EffaceableStorage - see iphone data protection project for how get this.

EMF!
• Data partition encryption key, encrypted with key 0x89B
• Format: length (0x20) + AES(key89B, emfkey)

### 2. DKey

设备密钥，Device key

- 功能描述：就是`NSProtectionNone` 类密钥的封装密钥，主要用于远程数据擦除
- 构造方式：原生随机数
- 加密方式：decrypt(Dkey, key 0x835)
- 储存位置：NAND闪存的可擦除区域

Dkey
• NSProtectionNone class key, wrapped with key 0x835
• Format: AESWRAP(key835, Dkey)

根据 iOS 的初期设计，`Class D`是所有用户数据文件的默认类型，其类密钥就是 DKey，即大多数文件的`per-file key`都通过 Dkey 进行封装，存储位置就是`systembag.kb`文件的 `Class 4` 字段。
但是，由于 DKey 并未将 Passcode 作为加密因子（仅由基于 UID 的 Key 0x835 提供保护），因此其加密强度难以抵抗暴力破解；进一步的，系统钥匙包仅仅是一个普通的数据文件，任何具有 root 级别访问权限的用户都可以轻松地获得并展开暴力破解，这显然是非常危险的。

从 iOS 5 开始，Apple 将 DKey 从设备密钥包中取出，存储位置转移到安全隔区专用的`Effaceable Storage`之中。

从 iOS 8 开始，为了应对 GrayKey 密码破解设备，Apple 决定将 Class C 作为默认类型，即在设备开机时，校验用户输入的 passcode，成功后即可自由访问（即使发生了锁屏事件，仍然可以访问数据文件）；但是如果设备关机，就必须在开机之后重新校验 passcode。
目前，Class D类的文件已经大为减少，这些文件在设备开机后就一直保持可访问状态。

### 3. Bag1

- 功能描述：系统密钥包（`/private/var/keybags/systembag.kb`）的文件级密钥，并包含一个初始向量IV
- 构造方式：原生随机数
- 加密方式：**不加密**。系统密钥包实际是一个数据文件，保存的Class Key还有一层加密
- 储存位置：NAND闪存的可擦除区域

系统密钥包有效负载密钥 （+初始化向量）。未加密地存储在可擦除区域。

BAG1
• System keybag payload key
• Format : magic (BAG1) + IV + Key
• Read from userland by keybagd to decrypt systembag.kb
• Erased at each passcode change to prevent attacks on previous keybag

## 五、File System 中的密钥

data partition 每个文件创建时，Data Protection会创建一个256-bit key (“per-file” key) ，然后使用这个密钥借助硬件 AES engine。加密模式通常是AES CBC mode（不懂的看密码学）. 既然是CBC自然需要IV，IV怎么来， 用文件的块偏移计算LFSR（ linear feedback shift register (LFSR) calculated with the block offset into the file）, IIV也是加密存储，encrypted with the SHA-1 hash of the per-file key.

per-file key是实际用来加密文件的，那它也得被保护啊。这就得分好几种条件了。
然后，加密的 per-file key is stored in the file’s metadata。.

然后file’s metadata又被加密了，The metadata of all files in the file system are encrypted with a random key, which is created when iOS is first installed or when the device is wiped by a user. The file system key is stored in Effaceable Storage. 其实这个加密价值不大。主要是为了销毁密钥方便。ios界面上操作 “Erase all content and settings” option,就是干掉了这个密钥。最终无法获得加密的per-file key。per-file key到底被谁加密，环境很多。根据不同的环境使用不用的 class key。 class key 又被 hardware UID 保护，有些情况下是通过user’s passcode. 如果你的数据相关联pin，那就得靠他。这非常重要！！！

多层密钥架构还有一个好处，就是对加密的内容更换密钥，无需重新加解密数据，那样效率很低，而只需更换加密key。

有密碼保護與無密碼保護的iTunes備份最大的差異是在Keybag是否被UID Key加密的狀態下儲存於iTunes備份擋內。若iTunes備份檔沒有密碼保護，Keybag會被UID的金鑰加密後儲存於備份檔內(Keychain也是)，這代表這個備份檔案是無法恢復到其他的iOS裝置，僅能恢復到原有備份出來的裝置(因為每個iOS的UID皆不同)。若iTunes備份檔有密碼保護，則Keybag會用iTunes設定的密碼採PBKDF2加密函式反覆運算1千萬次來保護 Keybag ，因Keybag未與UID綁定，當升級新的iOS裝置時就必須採用密碼保護的iTunes備份檔才可進行資料移轉。

有一个特定的储物柜叫做BAGI，它包含一个加密密钥，用来加密所谓的系统密钥包。密钥包包含许多加密“类密钥”，最终保护文件系统中的文件;它们在不同的时间被锁定和解锁，这取决于用户的活动。这让开发人员可以选择在设备被锁定时是否锁定文件，或者在输入密码后保持解锁，等等。文件系统上的每个文件都有自己的随机文件密钥，该密钥使用密钥包中的类密钥进行加密。密钥包的密钥是由BAGI储物柜中的密钥和用户的PIN的组合加密的。

系统会使用文件系统密钥(the file system key)解密文件的元数据， 以显露出封装的文件独有密钥和表示它受哪个类保护的记号
用于加密每个文件的元数据的密钥， 包括其类密钥。 存储在可擦除存储器中， 用于实现快速擦除， 并非用于保密目的。
加密的文件系统密钥还会使用储存在可擦除存储器中的“可擦除密钥”( an “effaceable key”) 封装，或者使用受安全隔区反重放机制保护的媒介密钥封装密钥（a media key-wrapping key）进行封装。
> the Secure Storage Component’s unique cryptographic key
此密钥不会提供数据的额外机密性。 相反，它可以根据需要快速抹掉。

> 数据宗卷文件系统中所有文件的元数据都使用随机宗卷密钥(a random volume key)进行加密， 该密钥在首次安装操作系统或用户擦除设备时创建。
> 此密钥由密钥封装密钥(a key wrapping key)加密和封装， 密钥封装密钥由安全隔区长期储存， 只在安全隔区中可见。 每次用户抹掉设备时， 它都会发生变化。
> 在 A9 （及后续型号） SoC 上， 安全隔区依靠由反重放系统支持的熵来实现可擦除性， 以及保护其他资源中的密钥封装密钥。 有关更多信息， 请参阅安全非易失性存储器。

![更细架构](arch2.png)

- 每个文件创建时会生成一个`Per-file Key`，在文件写入闪存时通过硬件用`AES-XTS`算法加密
- `Per-file Key`存储在文件元数据 Metadata 中，被`Class Key`和`Filesystem Key`加密
- `Class Key`和`Filesystem Key`都被硬件密钥`UID`保护
- `Class Key`还被用户密码`Passcode`保护

实际上，Apple就是在 HFS+ 文件系统的 inode 中增加了一个`cprotect` 字段：

- 如果数据保护级别是 A / B / C，存储以`Class Key`加密的`Per-file Key`
- 如果数据保护级别是 D，存储加密后的`DKey`(Device Key，正好也是D)
- 如果设置为`discarded`，则意味着密码失效，文件就永远无法打开，也是数据被抹去了...

## 六、xART key

一组为具有反重放功能 (基于物理储存架构) 的安全隔区提供加密且经认证的永久储存区的服务。

扩展反重放技术的密钥（eXtended Anti-Replay Technology key），就是第二代安全存储组件的**唯一加密密钥**.
后续苹果为了封堵各种暴力猜测Passcode的方法，在64位设备的Secure Enclave中增加了定时器，针对尝试密码的错误次数，增加尝试的延时，即使断电重启也无法解决。

## 七、SKP Key

在支持数据保护的 Apple 设备上， 密钥加密密钥 (KEK) 既受系统上软件测量值的保护 （或密封）， 又与只能从安全隔区获得的 UID 绑定。 在搭载 Apple 芯片的 Mac 上， 对 KEK 的保护通过整合有关系统安全性策略的信息进一步得到了加强， 因为 macOS 支持其他平台不支持的关键安全性策略更改 （例如， 停用安全启动或 SIP）。 在搭载 Apple 芯片的 Mac 上， 由于文件保险箱的实施使用数据保护 （C 类）， 此保护涵盖文件保险箱密钥。
On Apple devices that support Data Protection, the key encryption key (KEK) is protected (or sealed) with measurements of the software on the system, as well as being tied to the UID available only from the Secure Enclave.

xART An abbreviation for eXtended Anti-Replay Technology.

![SKP](SKP.png)

SKP = 密封密钥保护，也称为操作系统绑定密钥
KEK = key encryption key，密钥加密密钥
VEK = volume encryption key，卷宗加密密钥
xART key = eXtended Anti-Replay Technology key，扩展反重放技术的密钥，就是第二代安全存储组件的**唯一加密密钥**

SMRK = the crypto-hardware-derived System Measurement Root Key (SMRK)，系统测量根密钥，从加密硬件派生
SMDK = the system measurement device key，系统测量设备密钥，

PDK = Passcode-derived key，用户密码派生密钥

> 在 Apple A10 及后续型号的 SoC 上， PKA 支持操作系统绑定密钥， 也称为密封密钥保护 (SKP)。
> 这些密钥基于设备 UID 和设备上所运行 sepOS 的哈希值组合生成。
> 哈希值由安全隔区 Boot ROM 提供， 在Apple A13 及后续型号的 SoC 上， 则由安全隔区启动监视器提供。
> 这些密钥还用于在请求特定 Apple 服务时验证 sepOS 版本， 以及用于在系统发生重大更改而未获得用户授权时通过协助阻止访问密钥材料来提高受密码保护数据的安全性。

LLB - Low Level Bootloader，底层引导载入程序

在具有两步启动架构的 Mac 电脑上，LLB 包含由 Boot ROM 调用的代码，该代码随后会载入 iBoot，成为安全启动链的一环。

![搭载 Apple 芯片的 Mac 开机时的启动过程步骤](mac-boot.png)

如果攻击者能够意外地更改任何上述测量的固件、 软件或安全性配置组件， 则也会修改储存在硬件寄存器中的测量值。 测量值的修改会导致从加密硬件派生的系统测量根密钥 (SMRK) 派生出不同的值， 从而有效破坏密钥层 级的封章。 这将导致无法访问系统测量设备密钥 (SMDK)， 从而导致无法访问 KEK， 因此也无法访问数据。

但是， 系统在未受到攻击时， 必须容纳合法的软件更新， 这些更新会更改固件测量值和 LocalPolicy 中的 nsih 字段， 以指向新的 macOS 测量值。 在其他尝试整合固件测量值但没有已知真实来源的系统中， 用 户将被要求停用安全性， 更新固件后重新启用安全性， 以便捕获新的测量基线。 这大大增加了攻击者在软件 更新期间篡改固件的风险。 Image4 清单包含所需的所有测量值， 这对系统很有帮助。 正常启动期间如果测 量值匹配， 使用 SMRK 解密 SMDK 的硬件也可以将 SMDK 加密为所建议的将来的 SMRK。 通过指定软 件更新后预期的测量值， 硬件可以加密在当前操作系统中可访问的 SMDK， 以便在将来的操作系统中仍可 访问。 同样地， 当客户在 LocalPolicy 中合法更改其安全性设置时， 必须根据 LLB 下次重新启动时计算的 LocalPolicy 测量值， 将 SMDK 加密为将来的 SMRK。

> Image4 是经 ASN.1（抽象语法标记一）DER 编码的数据结构格式，用于描述 Apple 平台上有关安全启动链对象的信息。

---

![基本结构](arch1.png)
![简化架构](arch0.jpeg)

Data Wipe，数据擦除的工作步骤
Operation
• mobile_obliterator daemon
• Erase DKey by calling MKBDeviceObliterateClassDKey
• Erase EMF key by calling selector 0x14C39 in EffacingMediaFilter service
• Reformat data partition
• Generate new system keybag
• High level of confidence that erased data cannot be recovered




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

## 附录一：一些有趣的知识

### UDID - Unique Device Identifier，Apple设备的唯一识别号

UDID (Unique Device Identifier) 是一个由40个字符组成的十六进制序列，用于唯一标识一台苹果设备。它相当于设备的指纹，使得软件开发者和服务提供商能够提供个性化的设备识别服务。
UDID 可以被设备用户查询，例如 Mac 笔记本用户可以在`系统信息-硬件概览`中查看硬件 UDID。
![UDID](UDID.png)

构造算法：`UDID = SHA1(序列号 + ECID + Lowcase (WiFi MAC地址) + Lowcase(蓝牙 MAC地址))`

UDID 由于其唯一性，如果被泄露，恶意方可能会跟踪用户的活动或跨应用行为。因此，保护 UDID，以及使用它时要遵循隐私法和透明度原则是必要的。

---

NAND Key：负责加密GPT分区表和系统分区表，也被称为媒体密钥（media key）
文件系统密钥（f65dae950e906c42b254cc58fc78eece）：用于加密分区表和系统分区（图中称为“NAND 密钥”）
元数据密钥（92a742ab08c969bf006c9412d3cc79a5）：加密NAND元数据（vfl / ftl上下文和索引页面）

![文件保险箱](FileVault.png)

### 2. 遗留问题

1. 用户修改passcode后，需要重新封装metadata中所有的per-file key，似乎计算量也不小啊！

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