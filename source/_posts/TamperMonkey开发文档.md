---
title: TamperMonkey开发文档
date: 2020-12-03 18:26:12
tags:
---

## TamperMonkey概述

## 用户脚本标记头

- @name
- @namespace
- @version
- @author
- @description
- @homepage, @homepageURL, @website and @source
- @icon, @iconURL and @defaulticon
- @icon64 and @icon64URL
- @updateURL
- @downloadURL
- @supportURL
- @include
- @match
- @exclude
- @require
- @resource
- @connect
- @run-at
- @grant
- @noframes
- @unwrap
- @nocompat

## 应用编程接口

### 通用类

- GM_log(message)
- GM_info

### 本地存储类

- GM_setValue(name, value)
- GM_getValue(name, defaultValue)
- GM_deleteValue(name)
- GM_listValues()
  
— GM_addValueChangeListener(name, function(name, old_value, new_value, remote) {})
  对某个存储变量设置侦听器，并返回侦听器ID，通常用于不同浏览器选项卡的脚本之间进行通信，参考[开发示例](https://blog.csdn.net/weixin_42067967/article/details/105863853)。
  其中，`name`是观察到的变量的名称，回调函数的`remote`参数显示这个值是在另一个选项卡的实例中修改的`true`,还是在这个脚本实例中修改的`false`。
  
- GM_removeValueChangeListener(listener_id)
  删除上面设置的变量侦听器。

### 资源配置类

- GM_getResourceText(name)
- GM_getResourceURL(name)
- GM_registerMenuCommand(name, fn, accessKey)
- GM_unregisterMenuCommand(menuCmdId)

### 外部接口类

- GM_openInTab(url, options), GM_openInTab(url, loadInBackground)
- GM_xmlhttpRequest(details)
- GM_download(details), GM_download(url, name)
- GM_getTab(callback)
- GM_saveTab(tab)
- GM_getTabs(callback)
- GM_setClipboard(data, info)

### DOM管理类

- unsafeWindow
    默认TM可以访问DOM，但不可以直接调用页面javascript函数。
    如果头文件包含`// @grant unsafeWindow`，就可以通过`safeWindow`对象提供对页面javascript函数和变量的完全访问
- GM_addStyle(css)
- Subresource Integrity
- GM_notification(details, ondone), GM_notification(text, title, image, onclick)
- <><![CDATA[your_text_here]]></>

## 参考文献

- [TM官方文档](https://www.tampermonkey.net/documentation.php)
- [TM中文参考文档](https://www.cnblogs.com/grubber/p/12560522.html)
- 