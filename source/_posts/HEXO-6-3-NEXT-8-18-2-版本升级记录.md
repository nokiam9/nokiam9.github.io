---
title: HEXO 6.3 & NEXT 8.18.2 版本升级记录
date: 2023-10-18 00:23:48
tags:
---

原本 HEXO v5.4.2 基于 node v12 版本，但最近个别组件出现安全告警要求强制升级，但新版本的组件依赖 node v14 版本，只好再次升级。
![version](version.jpg)

当前，node 的最新版本 v20，LTS 版本 v18。Hexo 最新版本 v6.3，Next 主题最新版本 v8.18.2。

## package.json 配置信息

删除依赖库目录 `node_modules/` ，修改安装配置文件，然后重新`npm install`

```yaml
{
  "name": "hexo-site",
  "version": "0.0.0",
  "private": true,
  "scripts": {
    "build": "hexo generate",
    "clean": "hexo clean",
    "deploy": "hexo deploy",
    "server": "hexo server"
  },
  "hexo": {
    "version": "6.3.0"
  },
  "dependencies": {
    "@next-theme/plugins": "^8.18.2",           # Next主题的对应插件
    "hexo": "^6.3.0",                           # 主服务
    "hexo-auto-excerpt": "^1.1.0",              # 首页显示文章摘要，而非全文
    "hexo-deployer-git": "^4.0.0",              # git 远程发布
    "hexo-filter-mermaid-diagrams": "^1.0.5",   # mermaid 流程图的插件
    "hexo-generator-archive": "^2.0.0",         # 生成归档信息
    "hexo-generator-category": "^2.0.0",        # 生成分类信息
    "hexo-generator-index": "^3.0.0",           # 生成目录信息
    "hexo-generator-searchdb": "^1.4.1",        # 生成本地搜索数据库
    "hexo-generator-tag": "^2.0.0",             # 生成标记信息
    "hexo-renderer-ejs": "^2.0.0",              # EJS 渲染引擎，支持 v3 版本
    "hexo-renderer-marked": "^6.1.1",           # Markdown 渲染引擎
    "hexo-renderer-stylus": "^3.0.0",           # Stylus CSS 解析引擎
    "hexo-server": "^3.0.0",                    # 本地 Web 服务器
    "hexo-theme-next": "^8.18.2",               # Next 主题
    "hexo-word-counter": "0.1.0"                # 字数统计的插件
  }
}
```

## _config.yml 配置信息

注意需修改 hexo 的主配置文件！然后，根据需要相应调整 Next 的配置文件。

```yaml
post_asset_folder: true
marked:
  prependRoot: true
  postAsset: true
  dompurify: false  # 默认启用HTML标签净化，将导致markdown代码渲染失败
```

## 更新说明

1. 出于 HTML 安全考虑，新版本的[hexo-renderer-marked](https://github.com/hexojs/hexo-renderer-marked) 嵌入了[DOMPurify](https://github.com/cure53/DOMPurify)，但经常造成渲染异常，已建议采用新的插件 [hexo-renderer-markdown-it](https://github.com/hexojs/hexo-renderer-markdown-it/)
2. [hexo-auto-excerpt](https://github.com/ashisherc/hexo-auto-excerpt)插件通常用于显示文章摘要，已改为独立安装插件，而非进行配置；但由于 HTML 难以准确截断，推荐采用 <!--more--> 标签在文章内部进行显式截断
3. Hexo 默认安装了 hexo-renderer-marked 和 hexo-renderer-ejs，因此你不仅可以用 Markdown 写作，你还可以用 EJS 写作。如果你安装了 hexo-renderer-pug，你甚至可以用 Pug 模板语言书写文章。只需要将文章的扩展名从 md 改成 ejs，Hexo 就会使用 hexo-renderer-ejs 渲染这个文件，其他格式同理。
4. 早期 Next 主题的版本库位于：[theme-next/hexo-theme-next](https://github.com/theme-next/hexo-theme-next)，由于开发团队的内部矛盾而长期无法更新...于是部分开发者又搞了一个新版本：[next-theme/hexo-theme-next](https://github.com/next-theme/hexo-theme-next)，这就是 5.x 版本升级到 8.x 版本的重大变换，主要改造点：
   - 将配置文件移动到最外层，使用`_config.next.yml`，目的是后续的主题更新只需`git pull`，不需要担心配置文件冲突或者丢失的问题
   - 把库文件独立出来：[@next-theme/plugins](https://github.com/next-theme/plugins)；该插件需要独立安装，且版本号务必相同
   - 模板格式从 swig 调整为 njk
   - 保留 git 安装方式，但也可以支持 npm

---

## 参考文献

- [Hexo-5.x 与 NexT-8.x 跨版本升级](https://www.imczw.com/post/tech/hexo5-next8-updated.html)
