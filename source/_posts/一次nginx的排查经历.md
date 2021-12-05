---
title: 一次nginx的排查经历
date: 2018-12-21 00:18:06
tags: [nginx]
categories: [第三方组件,nginx]
---

## 现象
配置nginx的https，但是修改配置文件之后一直无法访问。。。


## 排查步骤
刚开始以为是防火墙的原因，由于是阿里云的主机，所以直接登录云主机查看安全配置。一切都是OK的。

然后又查看主机自己的防火墙，由于是`centos7`，所以在此花了点时间，最后还是将443端口添加到了防火墙规则中，然后重启防火墙。。。

`https`访问网站，发现还是没反应。


这个时候就开始怀疑nginx的配置文件是不是哪里配置错误了，主要都是修改的 443 的服务
> 其实这时候思路已经错误了

后来通过`https`还是一直不能访问到



## 继续排查

当时想了下，索性直接修改 `80` 配置的`server`，然后发现修改之后还能正常访问，于是就推断可能是修改的`nginx`的配置文件并非是nginx所读取的。于时通过以下命令发现问题：
```shell
[root@iZ2lqpf5ei7560Z ~]# locate nginx.conf
/usr/local/nginx/conf/nginx.conf
/usr/local/nginx/conf/nginx.conf.default
/usr/local/nginx/nginx-1.10.1/conf/nginx.conf
```
可以看到我的服务器中出现了两个`nginx.conf`，最下面那一个是nginx安装文件中的配置文件，我之前一直改的就是这个文件



然后再查看`nginx`的服务
```shell
[root@iZ2lqpf5ei7560Z ~]# ps -ef|grep nginx
root      4154     1  0 Dec16 ?        00:00:00 nginx: master process ./nginx
root      4155  4154  0 Dec16 ?        00:00:00 nginx: worker process
root     10071 10026  0 00:15 pts/2    00:00:00 grep --color=auto nginx
[root@iZ2lqpf5ei7560Z ~]# 
```

看到这里就大概知道原因了。之前一直改的是编译文件中的配置文件，自然是不会生效的。

## 思考
这也为之后遇到问题的排查思路提供了一点警示，即遇到问题的时候不要慌张，首先一点点的梳理逻辑，然后再进行排查。
