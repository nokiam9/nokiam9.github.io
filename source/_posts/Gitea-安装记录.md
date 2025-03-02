---
title: Gitea 安装记录
date: 2025-03-02 20:43:33
tags:
---

## 一、概述

Gitlab 是私有软件仓库的标准解决方案，但是国内只能使用所谓的“极狐版”，因此想找一个替代方案，意外发现了 Gitea。作为一款优秀的开源软件，Gitea 功能简洁，资源消耗少（2G内存即可，树莓派都可运行），不过由于是个人开发者，版本更新比较慢，但也足够使用了。
官方主页：[https://docs.gitea.com/](https://docs.gitea.com/)
中文主页：[https://docs.gitea.cn/](https://docs.gitea.cn/)

当前最新版本是 1.23.1，二进制代码的下载页面是：[https://dl.gitea.com/gitea/1.23.1/](https://dl.gitea.com/gitea/1.23.1/)
docker 是更好的安装方式，在软件安装目录编辑如下`/opt/gitea/docker-compose.yml` 文件即可。

可以发现部署了两个容器：主应用 `gitea/gitea:1.23.1` 和后端数据库 `mysql:8`。
安装目录位于：`/opt/gitea`，包含两个子目录，分别用于 gitea 和 mysql 容器。

```yml
version: "3"

networks:
  gitea:
    external: false

services:
  server:
    image: docker.io/gitea/gitea:1.23.1
    container_name: gitea
    environment:
      - USER_UID=1000
      - USER_GID=1000
      - GITEA__database__DB_TYPE=mysql
      - GITEA__database__HOST=db:3306
      - GITEA__database__NAME=gitea
      - GITEA__database__USER=gitea
      - GITEA__database__PASSWD=gitea
    restart: always
    networks:
      - gitea
    volumes:
      - ./gitea:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "80:3000"
      - "222:22"
    depends_on:
      - db

  db:
    image: docker.io/library/mysql:8
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=gitea
      - MYSQL_USER=gitea
      - MYSQL_PASSWORD=gitea
      - MYSQL_DATABASE=gitea
    networks:
      - gitea
    volumes:
      - ./mysql:/var/lib/mysql
```

> 为了避免 GW 问题，实际安装是通过 Harbor-proxy 下载，再修改 tag 提供给 docker

## 二、配置文件

Gitea 启动成功后，自动生成配置文件 `/opt/gitea/gitea/gitea/conf/app.ini`。
参数说明：[https://docs.gitea.cn/administration/config-cheat-sheet](https://docs.gitea.cn/administration/config-cheat-sheet)
可以停止 docker 服务后，根据需要手工修改配置，然后重启 docker 服务生效。

- Git 提供 HTTP 协议，默认容器端口 3000，对应主机端口 80
- Git 提供 SSH 协议，容器端口 22，主机端口 222（为了避开主机自己使用的 SSH 端口）
- HTTPS 可以成功配置，但后续 git 很难处理自签名证书，因此放弃了
- LFS 用于存储大对象文件，例如图片、视频等二进制文件

```yml
PROTOCOL = http
ROOT_URL = http://gitea.lan
HTTP_PORT = 3000 

DISABLE_SSH = false
SSH_PORT = 22
SSH_LISTEN_PORT = 22
LFS_START_SERVER = true
LFS_JWT_SECRET = N_DStu6KTGmYUNGA0O9K1ULA7hdTHK6vVEOlymvZeGE
OFFLINE_MODE = true
```

## 三、设置系统服务

编辑服务文件： `/etc/systemd/system/gitea.service`

```bash
[Unit]
Description=Harbor
After=docker.service systemd-networkd.service systemd-resolved.service
Requires=docker.service
Documentation=http://github.com/gitea/gitea

[Service]
Type=simple
Restart=on-failure
RestartSec=5
ExecStartPre=/usr/bin/docker-compose -f /opt/gitea/docker-compose.yml down
ExecStart=/usr/bin/docker-compose -f /opt/gitea/docker-compose.yml up
ExecStop=/usr/bin/docker-compose -f /opt/gitea/docker-compose.yml down

[Install]
WantedBy=multi-user.target
```

设置系统自启动

```bash
systemctl enable --now gitea
systemctl gitea status
```

## 四、Gitea 使用

访问方式： [http://gitea.lan](http://gitea.lan), 主机对外暴露 http 协议 80 端口。
注意！ssh 访问的端口是 222，而非标准的 22，因为 Gitea 实例运行在容器镜像，而非 host 操作系统。

Gitea 系统启动需要一段时间，可以跟踪容器日志查看，出现 http 侦听信息就说明启动成功。
![gitea](gitea.png)
第一次登陆需要注册 admin，以后就可以正常管理代码库了。

---

## 附录一：目录结构

`/opt/gitea/mysql` 是 Mysql 容器的数据目录，就不讨论了。
`/opt/gitea/gitea` 是 Gitea 容器的数据目录，下面三个子目录分别用于 git、ssh 和 用户代码库。

```console
/opt/gitea/gitea
├── git
│   ├── lfs
│   └── repositories
├── gitea
│   ├── actions_artifacts
│   ├── actions_log
│   ├── attachments
│   ├── avatars
│   ├── conf
│   ├── home
│   ├── indexers
│   ├── jwt
│   ├── log
│   ├── packages
│   ├── queues
│   ├── repo-archive
│   ├── repo-avatars
│   ├── sessions
│   └── tmp
└── ssh
```

## 附录二：INI 全量配置文件

配置文件位于：`$Gitea/gitea/conf/app.ini`

```yaml
APP_NAME = Gitea: Git with a cup of tea
RUN_MODE = prod
RUN_USER = git
WORK_PATH = /data/gitea

[repository]
ROOT = /data/git/repositories

[repository.local]
LOCAL_COPY_PATH = /data/gitea/tmp/local-repo

[repository.upload]
TEMP_PATH = /data/gitea/uploads

[server]
APP_DATA_PATH = /data/gitea
DOMAIN = gitea.lan
SSH_DOMAIN = gitea.lan

PROTOCOL = http
ROOT_URL = http://gitea.lan
HTTP_PORT = 3000 
# CERT_FILE = ../cert.pem
# KEY_FILE = ../key.pem

# config http transportfailled!
# REDIRECT_TO_OTHER_PORT = true
# PORT_TOREDIRECT = 3080

DISABLE_SSH = false
SSH_PORT = 22
SSH_LISTEN_PORT = 22
LFS_START_SERVER = true
LFS_JWT_SECRET = N_DStu6KTGmYUNGA0O9K1ULA7hdTHK6vVEOlymvZeGE
OFFLINE_MODE = true

[database]
PATH = /data/gitea/gitea.db
DB_TYPE = mysql
HOST = db:3306
NAME = gitea
USER = gitea
PASSWD = gitea
LOG_SQL = false
SCHEMA = 
SSL_MODE = disable

[indexer]
ISSUE_INDEXER_PATH = /data/gitea/indexers/issues.bleve

[session]
PROVIDER_CONFIG = /data/gitea/sessions
PROVIDER = file

[picture]
AVATAR_UPLOAD_PATH = /data/gitea/avatars
REPOSITORY_AVATAR_UPLOAD_PATH = /data/gitea/repo-avatars

[attachment]
PATH = /data/gitea/attachments

[log]
MODE = console
LEVEL = info
ROOT_PATH = /data/gitea/log

[security]
INSTALL_LOCK = true
SECRET_KEY = 
REVERSE_PROXY_LIMIT = 1
REVERSE_PROXY_TRUSTED_PROXIES = *
INTERNAL_TOKEN = eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYmYiOjE3Mzg2MDA1NTJ9.rRxDlAsDJa9dNc5Kdgi-GT7HkZpnQCk4tQ0dVj6JoPc
PASSWORD_HASH_ALGO = pbkdf2

[service]
DISABLE_REGISTRATION = false
REQUIRE_SIGNIN_VIEW = false
REGISTER_EMAIL_CONFIRM = false
ENABLE_NOTIFY_MAIL = false
ALLOW_ONLY_EXTERNAL_REGISTRATION = false
ENABLE_CAPTCHA = false
DEFAULT_KEEP_EMAIL_PRIVATE = false
DEFAULT_ALLOW_CREATE_ORGANIZATION = true
DEFAULT_ENABLE_TIMETRACKING = true
NO_REPLY_ADDRESS = noreply.localhost

[lfs]
PATH = /data/git/lfs

[mailer]
ENABLED = false

[openid]
ENABLE_OPENID_SIGNIN = true
ENABLE_OPENID_SIGNUP = true

[cron.update_checker]
ENABLED = false

[repository.pull-request]
DEFAULT_MERGE_STYLE = merge

[repository.signing]
DEFAULT_TRUST_MODEL = committer

[oauth2]
JWT_SECRET = 8m2nHq3h9dmVF9u3v5BYwJsRL3c9DN7XDLxctX20FeE

[actions]
ENABLED=true
```

---

## 官方网站

- [https://docs.gitea.cn/](https://docs.gitea.cn/)
- [使用 Docker 安装 Gitea](https://docs.gitea.cn/installation/install-with-docker)

## 参考文献

- [GitHub Actions 入门教程 - 阮一峰](https://www.ruanyifeng.com/blog/2019/09/getting-started-with-github-actions.html)
  