---
title: Python 经验汇编
date: 2024-09-22 00:01:27
tags:
---

## 1. 虚拟环境安装

参考 [Python教程 - 廖雪峰](https://liaoxuefeng.com/books/python/built-in-modules/venv/)。

注意找到正确的 Python 执行码版本，常见目录包括：

- Conda安装：`/usr/local/anaconda3/`
- 操作系统内置Python2：`/System/Library/Frameworks/Python.framework/Versions/`
- 官方Python3：`/Library/Frameworks/Python.framework/Versions/`

在当前的`.venv`子目录，安装虚拟环境

```console
python3 -m venv .venv
```

安装结果如下：

```console
sj@JiandeiMac signal-test % ls -l .venv
total 16
drwxr-xr-x  33 sj  staff  1056  9 21 23:56 bin
drwxr-xr-x   3 sj  staff    96  9 21 22:50 include
drwxr-xr-x   3 sj  staff    96  9 21 22:50 lib
-rw-r--r--   1 sj  staff   224  9 21 22:50 pyvenv.cfg
drwxr-xr-x   3 sj  staff    96  9 21 22:53 share
```

激活虚拟环境

```console
source .venv/bin/activate
```

显示当前已安装的软件包，默认只有pip，很干净！！！

```console
(.venv) sj@JiandeiMac aaa % pip3 list
Package Version
------- -------
pip     24.0
```

## 2. 设置国内安装源

### 一次性的替换源

```bash
pip3 install numpy -i https://pypi.tuna.tsinghua.edu.cn/simple
```

### 永久的替换源

查看pip的配置信息， `pip3 config list`，但是更好的是`pip3 config debug`

```console
(.venv) sj@JiandeiMac aaa % pip3 config debug
env_var:
env:
global:
  /Library/Application Support/pip/pip.conf, exists: False
site:
  /Users/sj/MyProject/aaa/.venv/pip.conf, exists: False
user:
  /Users/sj/.pip/pip.conf, exists: False
  /Users/sj/.config/pip/pip.conf, exists: True
    global.index-url: http://mirrors.aliyun.com/pypi/simple/
    install.trusted-host: mirrors.aliyun.com
```

配置文件名为`pip.conf`，路径依次为：系统安装目录---当前虚拟环境---当前用户归属目录

以`/Users/sj/.config/pip/pip.conf`为例，也可以用于`.venv/pip.conf`，其内容为：

```console
[global]
index-url = http://mirrors.aliyun.com/pypi/simple/

[install]
trusted-host = mirrors.aliyun.com
```

也可以使用命令行方式，但要注意实际修改了哪个配置文件：

```console
pip3 config set global.index-url http://mirrors.aliyun.com/pypi/simple/
pip3 config set install.trusted-host mirrors.aliyun.com
```

### 多个源的设置方法

注意，`index-url`字段只能有一个地址，其他在`extra-index-url`字段

```console
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
extra-index-url=
    https://pypi.mirrors.ustc.edu.cn/simple/
    https://pypi.douban.com/simple/
    https://pypi.python.org/simple/

[install]
trusted-host = tuna.tsinghua.edu.cn
    pypi.douban.com
    mirrors.ustc.edu.cn
    python.org
```

---

## 参考文献

- [Python教程 - 廖雪峰](https://liaoxuefeng.com/books/python/introduction/index.html)
- [pip 换源的注意事项](https://l-fay.github.io/2020/11/27/anaconda00/)

### 库函数

- [NumPy 教程 - 菜鸟教程](https://www.runoob.com/numpy/numpy-tutorial.html)
- [在 Python shell 中使用 Matplotlib](https://wizardforcel.gitbooks.io/matplotlib-user-guide/content/7.2.html)
- [Matplotlib教程](https://blog.csdn.net/sinat_41942180/article/details/134036932)
