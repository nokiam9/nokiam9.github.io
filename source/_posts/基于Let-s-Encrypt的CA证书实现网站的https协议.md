---
title: 基于Let's Encrypt的CA证书实现网站的https协议
date: 2020-06-06 16:56:40
tags:
---

支持HTTPS

随着大家对网络安全的重视，HTTPS 几乎已经成了网站的标配，很多接口都要求服务端必须为 HTTPS 协议，这也在无形中提高了我们的开发调试门槛。通过服务器的转发，这个问题也迎刃而解。
作为普通开发者，我们就不考虑购买昂贵的 HTTPS 证书了。好在良心产品 Let's Encrypt 从 ACME v2 开始已经支持泛解析了，于是 Let's Encrypt 一把梭搞定。这里强烈推荐国人开发的脚本 acme.sh，证书生成过程不再赘述。
有了证书，我们再配置一下提供 HTTPS 服务的 NGINX 配置，还可以给 HTTP 协议访问的页面做个 301 跳转：

```
server {
  listen 80;
  server_name "~^tunnel(?<port>\d+)\.gerald\.win$";

  location / {
    return 301 https://$host$request_uri;
  }
}

server {
  listen 443 ssl http2;
  server_name "~^tunnel(?<port>\d+)\.gerald\.win$";

  ssl_certificate      /home/gerald/ssl/fullchain.cer;
  ssl_certificate_key  /home/gerald/ssl/gerald.win.key;

  ssl_session_cache shared:SSL:1m;
  ssl_session_timeout  5m;

  ssl_ciphers  HIGH:!aNULL:!MD5;
  ssl_prefer_server_ciphers   on;

  location / {
    proxy_pass http://127.0.0.1:$port;
    proxy_set_header Host      $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_intercept_errors on;
  }
}
```
