---
title: 深入理解Dockerfile文件中的Entrypoint和Cmd命令
date: 2019-11-16 15:50:06
tags:
---

## CMD

[链接](https://docs.docker.com/engine/reference/builder/#cmd)

1. 主要目的

   - 提供默认的命令
   - 提供默认的参数，与ENTRYPOINT协同作用
   - 如果一个容器每次都要运行同一个可执行文件，推荐使用ENTRYPOINT
   - 该方式的命令，会被`docker run image cmd arg1 arg2`命令行方式覆盖
   - dockerfile 中最后一个CMD参数才会生效

2. 格式

exec form(推荐的方式): `CMD ["executable","param1","param2"]`
最终会被解析成json array， 必须有双引号
不提供shell环境，["echo", "$HOME"], HOME不会像shell一样解析
executable 要使用绝对路径
default parameters to ENTRYPOINT: CMD ["param1","param2"]
shell form: CMD command param1 param2 (shell form) 最好不要用
命令解析成： `/bin/sh -c command param1 param2`

## ENTRYPOINT

[链接](https://docs.docker.com/engine/reference/builder/#entrypoint)

1. 主要目的

   - 将镜像作为一个可执行文件，就像命令一样启动，无须指定命令：docker run -i -t --rm -p 80:80 nginx
   - ENTRYPOINT 命令参数的来源：
   dockerfile 中CMD的参数
   docker run命令行后面的参数列表。如`docker run <image> -d -c`, -d -c 将作为参数会给entrypoint。
   一般镜像都是以CMD多于ENTRYPOINT: 因为用cmd，docker run image cmd arg1 arg2, 可以很方便覆盖默认的命令，如果镜像是以ENTRYPOINT做为最终的执行命令，必须用 --entrypoint cmd 。 eg: d run --entrypoint /bin/sh -it xxx_image -c "echo hello"
   dockerfile 中最后一个ENTRYPOINT才会生效

2. 格式

    exec form(推荐方式): ENTRYPOINT ["executable", "param1", "param2"]
    shell form: ENTRYPOINT command param1 param2
    以/bin/sh -c 的形式运行命令
    会忽略signal信号，docker stop 无法停止
    坑较多，不要用
    ENTRYPOINT vs CMD

    [asd](https://docs.docker.com/engine/reference/builder/#understand-how-cmd-and-entrypoint-interact)

## 结论

用最简单的方式 CMD ["cmd", "arg1", "arg2"]
其他方式坑太多，细节太多
