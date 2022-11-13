---
title: Apple数据保护技术专题之二：密钥层级
date: 2022-11-13 14:34:19
tags:
---

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