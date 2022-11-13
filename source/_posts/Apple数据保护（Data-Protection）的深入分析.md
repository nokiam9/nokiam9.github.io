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

