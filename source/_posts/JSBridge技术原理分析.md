---
title: JSBridge技术原理分析
date: 2021-02-12 21:29:00
tags:
---

JSBridge 是一种JS 实现的Bridge，连接着桥两端的 Native 和 H5。 它在APP 内方便地让Native 调用JS，JS 调用Native ，是双向通信的通道。 JSBridge 主要提供了JS 调用Native 代码的能力，实现原生功能如查看本地相册、打开摄像头、指纹支付等。

## JS 调用 Native

## Native 调用 JS


{% asset_img life-cycle.png %}

---

## 参考文献

- [JSBridge原理的最佳教材](https://juejin.cn/post/6844903585268891662)
- [小白必看，JSBridge 初探](https://www.zoo.team/article/jsbridge)

- [Android安全开发之WebView中的地雷](https://www.zhihu.com/column/p/32146189)