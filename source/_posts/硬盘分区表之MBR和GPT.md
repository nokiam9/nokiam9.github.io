---
title: 硬盘分区表之MBR和GPT
date: 2020-06-20 22:39:03
tags:
---

## 名词解释

磁盘管理模式：

- MBR（Master Boot Record）：即硬盘的主引导记录分区列表，在主引导扇区，位于硬盘的cylinder 0， head 0， sector 1 （Sector是从1开始的）。

- GPT（GUID Partition Table）：即全局唯一标识分区列表，是一个物理硬盘的分区结构。它用来替代BIOS中的主引导记录分区表（MBR）。

主板接口标准：

- BIOS（Basic Input Output System）：它的全称应该是ROM－BIOS，意思是只读存储器基本输入输出系统。其实，它是一组固化到计算机内主板上一个ROM芯片上的程序，它保存着计算机最重要的基本输入输出的程序、系统设置信息、开机上电自检程序和系统启动自举程序。 其主要功能是为计算机提供最底层的、最直接的硬件设置和控制。

- UEFI（Unified Extensible Firmware Interface)：全称“统一的可扩展固件接口”， 是一种详细描述全新类型接口的标准。这种接口用于操作系统自动从预启动的操作环境，加载到一种操作系统上，从而使开机程序化繁为简，节省时间。

## 技术分析

MBR是传统的分区表类型，最大的缺点则是不支持容量大于2T的硬盘。GPT则弥补了MBR这个缺点，最大支持18EB的硬盘，是基于UEFI使用的磁盘分区架构。

传统BIOS主要支持MBR引导，UEFI则是取代传统BIOS，它加入了对新硬件的支持，其中就有2TB以上硬盘。

目前所有Windows系统均支持MBR，而GPT只有64位系统才能支持。BIOS只支持MBR引导系统，而GPT仅可用UEFI引导系统。正因为这样，现在主板大多采用BIOS集成UEFI，或UEFI集成BIOS，以此达到同时兼容MBR和GPT引导系统的目的。

UEFI启动引导系统的方法是查找硬盘分区中第一个FAT分区内的引导文件进行系统引导，这里并无指定分区表格式。所以U盘和移动硬盘可以用MBR分区表，创建一个FAT分区放置引导文件，从而达到可以双模式启动的目的。但需要注意的是，UEFI虽然支持MBR启动，但必须要有UEFI引导文件存放在FAT分区下；UEFI是无法使用传统MBR引导来启动系统的。

由于GPT引导系统的方式与MBR不同，故而使用传统系统安装办法（如Ghost、Win$Man等）会导致出现系统无法引导的现象。而且使用GPT引导系统的话，必要时还得调整主板设置，开启UEFI（大部分主板默认开启UEFI）。但是使用UEFI和GPT，只是支持大于容量2T的硬盘，并不会带来质的提升（开机硬件自检会稍微快了那么1、2秒）。所以，如果不用大于2T的硬盘做系统的话，就没必要使用UEFI。

- BIOS+MBR：这是最传统的，系统都会支持；唯一的缺点就是不支持容量大于2T的硬盘。

- BIOS+GPT：BIOS是可以使用GPT分区表的硬盘来作为资料盘的，但不能引导系统；若电脑同时带有容量小于2T的硬盘和容量大于2T的硬盘，小于2T的可以用MBR分区表安装系统，而大于2T的可以使用GPT分区表来存放资料。但系统须使用64位系统。

- UEFI+MBR：可以把UEFI设置成Legacy模式（传统模式）让其支持传统MBR启动，效果同BIOS+MBR；也可以建立FAT分区，放置UEFI启动文件来，可应用在U盘和移动硬盘上实现双模式启动。

- UEFI+GPT：如果要把大于2T的硬盘作为系统盘来安装系统的话，就必须如此。而且系统须使用64位系统，否则无法引导。但系统又不是传统在PE下安装后就能直接使用的，引导还得经过处理才行。

## 实际案例

Ubuntu安装完成后，主机硬盘的默认分区表为GPT格式（不是MBR格式），包含了两个分区，其中：

- `/dev/sda1`：用于EFI启动，512M
- `/dev/sda2`：Linunx系统盘

``` shell
root@nuc5i3:~# fdisk -l
Disk /dev/loop0: 89.1 MiB, 93417472 bytes, 182456 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sda: 111.8 GiB, 120034123776 bytes, 234441648 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: D8D94899-BC23-4D81-96EB-ECAFE4710A7E

Device       Start       End   Sectors   Size Type
/dev/sda1     2048   1050623   1048576   512M EFI System
/dev/sda2  1050624 234438655 233388032 111.3G Linux filesystem
```
