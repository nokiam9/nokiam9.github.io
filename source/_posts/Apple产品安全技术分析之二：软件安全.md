---
title: Apple产品安全技术分析之二：软件安全
date: 2022-07-09 21:55:39
tags:
---

## 附录三：Apple的安全漏洞事件

- 早期的A4及更老的芯片(iPhone4之前的设备包括iPhone4)存在bootrom漏洞，通过bootrom中的硬件漏洞获取设备的shell然后运行暴力破解程序。[A4后面的芯片目前没有公开的bootrom漏洞]
- iOS7中存在利用外接键盘可以暴力破解密码，甚至停用的设备也可以破解的漏洞。[该漏洞已经在iOS8中修复]
- iOS8的早期几个版本中存在密码尝试失败立即断电并不会增加错误计数的漏洞。[该漏洞已经修复]
- 2020 年秋天，苹果突然发布第二代安全隔区，并紧急升级 A12、A13 以及 S5 芯片，据认为 GrayKey 密码破解设备有关系，其采用暴力破解方式实现iPhone解锁。

![bootrom](bootrom.png)

---

## 参考文献

- [iPhone史诗级DFU漏洞分析](https://www.bilibili.com/read/cv9849473/)
- [Checkm8 漏洞研究](https://xuanxuanblingbling.github.io/ios/2020/07/10/checkm8/)
- [苹果超级大漏洞 BootROM 的说明及威胁评估](https://zhuanlan.zhihu.com/p/84925896)