---
title: TamperMonkey开发文档
date: 2020-12-03 18:26:12
tags:
---

## 一、概述

### TamperMonkey

Tampermonkey 是一款免费的浏览器扩展和最为流行的用户脚本管理器，它适用于 Chrome, Microsoft Edge, Safari, Opera Next, 和 Firefox。

虽然有些受支持的浏览器拥有原生的用户脚本支持，但 Tampermonkey 将在您的用户脚本管理方面提供更多的便利。 它提供了诸如便捷脚本安装、自动更新检查、标签中的脚本运行状况速览、内置的编辑器等众多功能， 同时Tampermonkey还有可能正常运行原本并不兼容的脚本。

### Chrome Extension

从本质上讲，TamperMonkey是一个用于管理Chrome Extension的软件。

我们经常说的 Chrome “插件”，其实不是真正意义上的 Chrome Plug-in，一般是指 Chrome Extension(简称“拓展”)。

- 扩展（Extension），指的是通过调用 Chrome 提供的 Chrome API 来扩展浏览器功能的一种组件，工作在浏览器层面，使用 HTML + Javascript 语言开发。
- 插件（Plug-in），指的是通过调用 Webkit 内核 NPAPI/PPAPI 来扩展内核功能的一种组件，工作在内核层面，理论上可以用任何一种生成本地二进制程序的语言开发，比如 C/C++、Delphi 等。比如 Flash player 插件，就属于这种类型。一般在网页中用 `<object>` 或者 `<embed>` 标签声明的部分，就要靠插件来渲染。

{% asset_img extension-architecture.png Chrome Extension的技术架构 %}

Chrome 拓展的 JS 主要可以分为这 5 类：injected script、content-script、popup js、background js 和 devtools js，

|JS种类|可访问的API|DOM访问情况|JS访问情况|直接跨域|
| :---: |:---: |:---: |:---: |:---: |
|injected script|和普通 JS 无任何差别，不能访问任何扩展 API|可以访问|可以访问|不可以|
|content script|只能访问 extension、runtime 等部分API|可以访问|不可以|不可以|
|popup js|可访问绝大部分 API，除了 devtools 系列|不可直接访问|不可以|可以|
|background js|可访问绝大部分 API，除了 devtools 系列|不可直接访问|不可以|可以|
|devtools js|只能访问 devtools、extension、runtime 等部分API|可以|可以|不可以|

## 二、用户脚本标记头

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

## 三、应用编程接口

### 通用类

- GM_log(message)
- GM_info

### 本地存储类

GM存储并不是浏览器的localStorage数据，而是TM自行定义的，仅在本TM内部有效。

- GM_setValue(name, value)
- GM_getValue(name, defaultValue)
- GM_deleteValue(name)
- GM_listValues()
  这个函数的返回值很奇葩！是一个包含所有name的数组，而不是value。
  而且，js中for循环中的自变量，如果母体是Array，自变量是计数器，而非数组的内容。
  
  ```js
  const names = GM_listValues();
  let rs = new Map();
  for (let i in names) rs.set(names[i], GM_getValue(names[i]));
  console.log('GM_listValues:', rs);
  ```

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

## 四、TamperMonkey的默认用户脚本

``` js
// ==UserScript==
// @name         New Userscript
// @namespace    http://tampermonkey.net/
// @version      0.1
// @description  try to take over the world!
// @author       You
// @match        http://*/*
// @grant        none
// ==/UserScript==

(function() {
    'use strict';

    // Your code here...
})();
```

---

``` console
--Start Firefox--
export DISPLAY=:0
source /etc/profile
/usr/bin/python3 -m webbrowser -t https://b2b.10086.cn/b2b/main/listVendorNotice.html?noticeType=1
␇
--end--
START /usr/bin/firefox "https://b2b.10086.cn/b2b/main/listVendorNotice.html?noticeType=1"
Failed to open connection to "session" message bus: /usr/bin/dbus-launch terminated abnormally with the following error:
 No protocol specified
Autolaunch error: X11 initialization failed.

Running without a11y support!
No protocol specified
Error: cannot open display: :0
xdg-open: no method available for opening 'https://b2b.10086.cn/b2b/main/listVendorNotice.html?noticeType=1'

```

---

## 参考文献

### 官方文档

- [TamperMonkey官方文档](https://www.tampermonkey.net/documentation.php)
- [Greasy Fork: 一个提供用户脚本的网站](https://greasyfork.org/zh-CN)
- [TamperMonkey中文参考文档](https://www.cnblogs.com/grubber/p/12560522.html)
- [ECMAScript 6 经典教程](https://es6.ruanyifeng.com/)
- [Chrome DevTools Extensions官方文档](https://developer.chrome.com/extensions/devtools)
- [Chrome DevTools Extensions中文文档](https://crxdoc-zh.appspot.com/extensions/devtools)
- [Chrome 内容安全策略（CSP）](https://crxdoc-zh.appspot.com/extensions/contentSecurityPolicy)
- [深入理解 Chrome DevTools](https://zhaomenghuan.js.org/blog/chrome-devtools.html)
  
### 开发案例

- [Chrome Extension开发基础知识](https://juejin.cn/post/6844904127932137485)
- [Content Security Policy 入门教程](http://www.ruanyifeng.com/blog/2016/09/csp.html)
- [chrome拓展开发实战：页面脚本的拦截注入](https://horve.github.io/2015/10/17/chrome-extension/)
- [基于TM插件的某个优秀图片爬虫](https://github.com/FoXZilla/Pxer/blob/master/README.zh.md)
- [GM_getTab开发示例](https://www.thinbug.com/q/52415273)
- [GM_addValueChangeListener开发示例](https://blog.csdn.net/weixin_42067967/article/details/105863853)
- [一个TM爬虫的粗糙示例](https://zhuanlan.zhihu.com/p/67221319)
- [常见跨域解决方案](https://juejin.cn/post/6844903575143841805)
