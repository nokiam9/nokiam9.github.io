---
title: Mongo数据库设置鉴权登录的方法
date: 2019-11-15 16:57:17
tags:
---

Mongo数据库默认采用免鉴权的登录方式，虽然很方便，但是有被劫持勒索的风险，为此最好采用鉴权登录的方法。 

## 配置初始化用户和口令

以[mongo:3.6镜像文件](https://github.com/docker-library/mongo/tree/master/3.6)为例，启动命令是`/usr/local/bin/docker-entrypoint.sh`。

分析该脚本文件可以发现其基本步骤是：

1. 下载Mongo数据库的代码，并以免鉴权方式启动。
2. 如果OS设置了环境变量`MONGO_INITDB_ROOT_USERNAME`和`MONGO_INITDB_ROOT_PASSWORD`，则新建数据库用户并设置权限规则。 
    注意：mongo初始化用户的角色级别为root，需要授权创建和修改database。
3. 重新启动数据库。 

## 以鉴权方式连接Mongo

- 以mongo shell为例，在命令行中带入用户名和密码，就可以以鉴权方式登录了。

    ``` console
    $ mongo -u username -p password --authenticationDatabase admin
    ```

- 如果采用pymong登录，示例代码为：
  
    ``` python
    import logging
    from pymongo import MongoClient

    uri = "mongodb://username:password@localhost:27017"
    client = MongoClient(uri)

    try:
        # Test with this ismaster command is cheap and does not require auth.
        client.admin.command('ismaster')
        logger.info(u"Connected to MongoDB, uri={0}.".format(uri))
    except:
        logger.error(u"Connect MongoDB sever failed and abort now! uri={0}.".format(uri))
        raise SyntaxError  
    ```

---

## 关于flask-mongoengine的疑难问题

## 现象描述

调用`flask-mongoengine`连接mongo，采用URI方式配置参数一直报各类鉴权错误

## 原因分析

根本原因是flask-pymongo的封装存在bug，其规则是：

- MONGO_DBNAME如果没有设置的话，用于验证的数据库就会被设置成app.name。
- 如果设置了MONGO_DBNAME，用于验证和连接的数据库都会变成MONGO_DBNAME。

所以经过分析，我们可以不使用MONGO_DBNAME，然后让DBNAME通过app.name来进行设置。

请看如下技术文档：

[MongoEngine关于connect的规定](http://docs.mongoengine.org/guide/connecting.html) ：

>- If the database requires authentication, username, password and authentication_source arguments should be provided.
>- Database, username and password from URI string overrides corresponding parameters in connect().

``` python
from mongoengine import connect

connect(
    db='test',
    username='user',
    password='12345',
    host='mongodb://admin:qwerty@localhost/production'
)
```

[Flask Mongoengine关于connect的规定](https://mongoengine-odm.readthedocs.io/guide/connecting.html):  

>- Uri style connections are also supported, just supply the uri as the host in the ‘MONGODB_SETTINGS’ dictionary with app.config.
>- Note that database name from uri has priority over name.

## 解决方法

1. 在使用Mongoengine方式连接Mongo数据库时，不要采用URI方式，而是单独设置每个参数;
2. 启用鉴权方式时，必须显式包含username、password、authentication_source。

示例代码如下：

- 在配置文件`settings.py`中设置Mongo的配置参数：

    ``` python
    import os

    MONGODB_SETTINGS = {
        'db': 'cmccb2b',
        'username': os.getenv('MONGODB_USERNAME'),
        'password': os.getenv('MONGODB_PASSWORD'),
        'host': os.getenv('MONGODB_HOST'),
        'port': int(os.getenv('MONGODB_PORT')),
        'connect': False,  # set for pymongo bug fix
        'authentication_source': 'admin', # set authentication source database， default is MONGODB_NAME
    }
    ```

- 在主入口`main.py`中启动flask：

    ``` python  
    from flask import Flask
    from flask_mongoengine import MongoEngine

    # 初始化app，原型在flask，并从settings.py中提取自定义的类属性，包括MongoDB配置，debug配置等
    app = Flask(__name__,
                static_folder='static',
                template_folder='templates')
    app.config.from_pyfile(filename='settings.py')

    # 初始化数据库连接db
    db = MongoEngine()  

    # 连接flask和mongoengine，注意db在models.py中初始化，参数设置已经从app.config中加载
    db.init_app(app)
    ```

- 相关示范案例，请参见[Mongo配置鉴权方式的经验](https://www.techcoil.com/blog/how-to-enable-authenticated-mongodb-access-for-flask-mongoengine-applications/)
- [Flask-Pymongo登陆验证问题小记](https://nladuo.github.io/2018/10/25/Flask-Pymongo%E7%99%BB%E9%99%86%E9%AA%8C%E8%AF%81%E9%97%AE%E9%A2%98%E5%B0%8F%E8%AE%B0/) 