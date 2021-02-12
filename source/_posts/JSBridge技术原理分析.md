---
title: JSBridge技术原理分析
date: 2021-02-12 21:29:00
tags:
---

## JSBridge的起源

`PhoneGap`（Codova 的前身）作为 Hybrid 鼻祖框架，是一个开源的移动开发框架，允许你用标准的web技术-HTML5,CSS3和JavaScript做跨平台的Hybird WebUI开发，应该是最先被开发者广泛认知的 JSBridge 的应用场景。而对于 JSBridge 的应用在国内真正兴盛起来，则是因为杀手级应用微信的出现。

JSBridge 是一种JS 实现的Bridge，连接着桥两端的 Native 和 H5。 简单来讲，它在APP 内方便地让Native 调用JS，JS 调用Native ，是双向通信的通道。 JSBridge 主要提供了JS 调用Native 代码的能力，实现原生功能如查看本地相册、打开摄像头、指纹支付等。

{% asset_img jsbridge.png %}

既然是『简单来讲』，那么 JSBridge 的用途肯定不只『调用 Native 功能』这么简单宽泛。实际上，JSBridge 就像其名称中的『Bridge』的意义一样，是 Native 和非 Native 之间的桥梁，它的核心是 构建 Native 和非 Native 间消息通信的通道，而且是双向通信的通道。

> 微信、头条等小程序基于 Web UI，但是为了追求运行效率，对 UI 展现逻辑和业务逻辑的 JavaScript 进行了隔离。

## JSBridge的实现原理

JavaScript 是运行在一个单独的 JS Context 中（例如，WebView 的 Webkit 引擎、JSCore）。由于这些 Context 与原生运行环境的天然隔离，我们可以将这种情况与 RPC（Remote Procedure Call，远程过程调用）通信进行类比，将 Native 与 JavaScript 的每次互相调用看做一次 RPC 调用。

在 JSBridge 的设计中，可以把前端看做 RPC 的客户端，把 Native 端看做 RPC 的服务器端，从而 JSBridge 要实现的主要逻辑就出现了：通信调用（Native 与 JS 通信） 和 句柄解析调用。（如果你是个前端，而且并不熟悉 RPC 的话，你也可以把这个流程类比成 JSONP 的流程）

通过以上的分析，可以清楚地知晓 JSBridge 主要的功能和职责，接下来就以 Hybrid 方案 为案例从这几点来剖析 JSBridge 的实现原理。

### JS 调用 Native

Hybrid 方案是基于 WebView 的，JavaScript 执行在 WebView 的 Webkit 引擎中。因此，Hybrid 方案中 JSBridge 的通信原理会具有一些 Web 特性。

#### 方式1：注入API

对于 iOS来说，

- UIWebView提供了`JavaScriptScore`方法，支持 iOS 7.0 及以上系统
- WKWebview提供了 `window.webkit.messageHandlers` 方法，支持 iOS 8.0 及以上系统。

对于Andriod来说，

- 4.2 之前，Android 注入 JavaScript 对象的接口是 `addJavascriptInterface`，但是这个接口有漏洞，可以被不法分子利用，危害用户的安全，
- 4.2 之后，Android引入新的接口 `@JavascriptInterface`以解决安全问题,所以 Android 注入对对象的方式是有兼容性问题的。

#### 方式2：拦截 URL SCHEME

`URL SCHEME`是一种类似于url的链接，是为了方便app直接互相调用设计的，形式和普通的 url 近似，主要区别是 protocol 和 host 一般是自定义的，例如: `qunarhy://hy/url?url=ymfe.tech`，protocol 是 qunarhy，host 则是 hy。

拦截 URL SCHEME 的主要流程是：Web 端通过某种方式（例如 iframe.src）发送 URL Scheme 请求，之后 Native 拦截到请求并根据 URL SCHEME（包括所带的参数）进行相关操作。

在实践过程中，这种方式有一定的缺陷：

1. 使用 iframe.src 发送 URL SCHEME 会有 url 长度的隐患。
2. 创建请求，需要一定的耗时，比注入 API 的方式调用同样的功能，耗时会较长。

但是这种方式的最重要优势是跨平台兼容性，尤其是**支持 iOS6**，但考虑到终端覆盖率，已经不是主流方式。

#### 方式3：重写 prompt 等原生 JS 方法

WebView有一个方法，叫`setWebChromeClient`，可以设置`WebChromeClient`对象，而这个对象中有三个方法，分别是`onJsAlert`,`onJsConfirm`,`onJsPrompt`，当js调用window对象的对应的方法，即`window.alert`，`window.confirm`，`window.prompt`，WebChromeClient对象中的三个方法对应的就会被触发，我们是不是可以利用这个机制，自己做一些处理呢？答案是肯定的。

由于拦截上述方法会对性能造成一定影响，因此需要选择使用频率较低的方法，而在Android中，相比其它几个方法，几乎不会使用到`prompt`方法，因此占用`prompt`是最佳方案。

### Native 调用 JS

Native 调用 JS 比较简单，只要 H5 将 JS 方法暴露在 Window 上给 Native 调用即可。

Android 中主要有两种方式实现。

- 在 4.4 以前，通过 `loadUrl` 方法，执行一段 JS 代码来实现。
    loadUrl 方法使用起来方便简洁，但是效率低无法获得返回结果且调用的时候会刷新 WebView 。
- 在 4.4 以后，可以使用 `evaluateJavascript` 方法实现。
    该方法效率高获取返回值方便，调用时候不刷新 WebView，但是只支持 Android 4.4+。

## JSBridge 如何引用

对于 JSBridge 的引用，常用有两种方式，各有利弊。

### 方式1：由 Native 端进行注入

注入方式和 Native 调用 JavaScript 类似，直接执行桥的全部代码。

它的优点在于：桥的版本很容易与 Native 保持一致，Native 端不用对不同版本的 JSBridge 进行兼容；

它的缺点是：注入时机不确定，需要实现注入失败后重试的机制，保证注入的成功率，同时 JavaScript 端在调用接口时，需要优先判断 JSBridge 是否已经注入成功。

### 方式2：由 JavaScript 端引用

与由 Native 端注入正好相反，它的优点在于：JavaScript 端可以确定 JSBridge 的存在，直接调用即可；

缺点是：如果桥的实现方式有更改，JSBridge 需要兼容多版本的 Native Bridge 或者 Native Bridge 兼容多版本的 JSBridge。

---

## 参考文献

- [JSBridge原理的最佳教材](https://juejin.cn/post/6844903585268891662)
- [小白必看，JSBridge 初探](https://www.zoo.team/article/jsbridge)
- [微信小程序weapp的底层实现原理](https://developers.weixin.qq.com/community/develop/doc/d1421cd729a51548672430e544c458b2)
- [Android JSBridge的原理与实现](https://blog.csdn.net/sbsujjbcy/article/details/50752595)
- [Android安全开发之WebView中的地雷](https://www.zhihu.com/column/p/32146189)
