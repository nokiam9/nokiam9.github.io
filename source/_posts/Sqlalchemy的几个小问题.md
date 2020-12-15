---
title: Sqlalchemy的几个小问题
date: 2020-12-16 07:51:16
tags:
---

## 改写为SQLALCHEMY

## 注意事项

- pyecharts == 0.5.3

- python3不再支持 mysql库了，需要使用pymysql；
  修改为engine=create_engine('mysql+pymysql://root:password@localhost:3306/test')就可以了

``` console
>>> str
'2020-12-20'
>>> atime = time.mktime(time.strptime(str, '%Y-%m-%d'))
>>> atime
1608393600.0
>>> time.ctime(atime)
'Sun Dec 20 00:00:00 2020'
```

`now.timestamp()`

``` sql
mysql> show variables like 'character_set_%';
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | latin1                     |
| character_set_connection | latin1                     |
| character_set_database   | latin1                     |
| character_set_filesystem | binary                     |
| character_set_results    | latin1                     |
| character_set_server     | latin1                     |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
8 rows in set (0.00 sec)

mysql> status
--------------
mysql  Ver 14.14 Distrib 5.7.32, for Linux (x86_64) using  EditLine wrapper

Connection id:          247
Current database:       cmccb2b
Current user:           root@localhost
SSL:                    Not in use
Current pager:          stdout
Using outfile:          ''
Using delimiter:        ;
Server version:         5.7.32 MySQL Community Server (GPL)
Protocol version:       10
Connection:             Localhost via UNIX socket
Server characterset:    latin1
Db     characterset:    latin1
Client characterset:    latin1
Conn.  characterset:    latin1
UNIX socket:            /var/run/mysqld/mysqld.sock
Uptime:                 2 days 3 hours 11 min 59 sec

Threads: 9  Questions: 5348  Slow queries: 0  Opens: 294  Flush tables: 1  Open tables: 182  Queries per second avg: 0.029
--------------
```

SQLALCHEMY_DATABASE_URI = 'mysql+pymysql://root:123456@localhost:3306/cmccb2b?charset=utf8'

## 参考文献

- [Flask-SQLAlchemy Home](https://flask-sqlalchemy.palletsprojects.com/en/2.x/)
- [SQLAlchemy中文文档](https://www.osgeo.cn/sqlalchemy/orm/index.html)
- [SQLAlchemy Home 文字真烂](https://docs.sqlalchemy.org/en/13/core/engines.html#database-urls)
- [sqlalchemy-paginator Home](https://github.com/ahmadjavedse/sqlalchemy-paginator)

- [使用sqlalchemy进行数据库操作示例1](https://my.oschina.net/u/4268222/blog/3515823)
- [sqlalchemy的使用示例2](https://jingniao.github.io/2016/11/26/sqlalchemy-use-start/)
- [SQLAlchemy ORM的简明教程](https://www.jianshu.com/p/0d234e14b5d3)
- [Flask-SQLAlchemy的高级示例（英文）](https://hackersandslackers.com/flask-sqlalchemy-database-models/)

- [MySQL字符集设置大全](https://www.cnblogs.com/chyingp/p/mysql-character-set-collation.html)

---

- [asynhttp Home](https://docs.aiohttp.org/en/stable/web_quickstart.html#variable-resources)
- [Jinja2 中文手册](https://www.csdn.net/handbook/jinja/jinja2/templates.html#variables)

- [四种常见的 POST 提交数据方式](https://imququ.com/post/four-ways-to-post-data-in-http.html)