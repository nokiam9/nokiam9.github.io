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

GM存储并不是浏览器的localStorage数据，而是TM自行定义的，仅在本TM内部有效。

- GM_setValue(name, value)
- GM_getValue(name, defaultValue)
- GM_deleteValue(name)
- GM_listValues()

`GM_addValueChangeListener`可以对某个存储变量设置侦听器，用于不同浏览器选项卡的脚本之间的通信。
其中，回调函数的`remote`参数显示这个值是在另一个选项卡的实例中修改的`true`,还是在这个脚本实例中修改的`false`。

- GM_addValueChangeListener(name, function(name, old_value, new_value, remote) {})
- GM_removeValueChangeListener(listener_id)

### 配置信息类

- GM_getResourceText(name)
- GM_getResourceURL(name)
- GM_registerMenuCommand(name, fn, accessKey)
- GM_unregisterMenuCommand(menuCmdId)

### 外部接口类

- GM_openInTab(url, options), GM_openInTab(url, loadInBackground)
  使用此url打开一个新选项卡，和`window.open()` 功能类似。
  注意，其返回值并不是标准的`window`对象，而是一个包含函数close、侦听器onclosed、closed标记的奇怪对象。
- GM_xmlhttpRequest(details)
  重要！！！由于TM运行在浏览器中，无法访问host数据，只能通过XHR保存爬取来的数据。
- GM_download(details), GM_download(url, name)
- GM_getTab(callback)
- GM_saveTab(tab)
- GM_getTabs(callback)
- GM_setClipboard(data, info)

### DOM资源类

- unsafeWindow
    默认TM可以访问DOM，但不可以直接调用页面javascript函数。
    如果头文件包含`// @grant unsafeWindow`，就可以通过`safeWindow`对象提供对页面javascript函数和变量的完全访问
- GM_addStyle(css)
  将给定的样式添加到文档并返回注入的样式元素。
- Subresource Integrity
  @resource 和 @require 标记 URL 的哈希组件可用于此目的。
- GM_notification(details, ondone), GM_notification(text, title, image, onclick)
  显示 HTML5 桌面通知和/或突出显示当前选项卡。
- <><![CDATA[your_text_here]]></>
  Tampermonkey支持这种存储元数据的方式。TM尝试自动检测脚本是否需要启用此兼容性选项。

## 参考文献

- [TM官方文档](https://www.tampermonkey.net/documentation.php)
- [TM中文参考文档](https://www.cnblogs.com/grubber/p/12560522.html)
- [ECMAScript 6 经典教程](https://es6.ruanyifeng.com/)
- [某个基于TM插件的图片爬虫](https://github.com/FoXZilla/Pxer/blob/master/README.zh.md)
- [GM_getTab开发示例](https://www.thinbug.com/q/52415273)
- [GM_addValueChangeListener开发示例](https://blog.csdn.net/weixin_42067967/article/details/105863853)。
