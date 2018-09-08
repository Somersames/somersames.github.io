---
title: Linux上安装rabbitmq遇到的一些问题
date: 2018-08-29 23:21:20
tags: [Linux,rabbitmq]
categories: Linux
---
# 在安装elixir的时候erlang，安装了错误的包
错误记录如下：
```log
{"init terminating in do_boot",{'cannot get bootfile','no_dot_erlang.boot'}}
init terminating in do_boot ({cannot get bootfile,no_dot_erlang.boot})
```
此时的错误是在安装erlang的时候安装了错误的erlang，正确的需要安装的是`esl-erlang`，详情如下：
> The "esl-erlang" package is a file containg the complete installation: it includes the Erlang/OTP platform and all of its    applications. The "erlang" package is a frontend to a number of smaller packages. Currently we support both "erlang" and "esl-erlang". Note that the split packages have multiple advantages:
seamless replacement of the available packages,
other packages have dependencies on "erlang", not "esl-erlang",
if your disk-space is low, you can get rid of some unused parts; "erlang-base" needs only ~13MB of space.

也就是说相较于`erlang`，`esl-erlang`的安装包是包含了所有的组件的。

# rabbitmq和erlang的cookies不一致，导致启动不了
启动的日志如下：
```java
attempted to contact: [rabbit@izm5e0h94dt7do1kplgd15z]

rabbit@izm5e0h94dt7do1kplgd15z:
  * connected to epmd (port 4369) on izm5e0h94dt7do1kplgd15z
  * epmd reports node 'rabbit' running on port 25672
  * TCP connection succeeded but Erlang distribution failed

  * Authentication failed (rejected by the remote node), please check the Erlang cookie

  current node details:
  - node name: 'rabbitmq-cli-56@izm5e0h94dt7do1kplgd15z'
  - home dir: /root
  - cookie hash: ajtINcxQAbART7QakzjkSg==

```
在搜索中发现了两个链接：[StackOverFlow](https://stackoverflow.com/questions/47893899/authentication-failed-rejected-by-the-remote-node-please-check-the-erlang-coo?rq=1)
、[Rabbitmq官方文档](http://www.rabbitmq.com/cli.html)

其中在官网中看到了介绍：
> Linux, MacOS, *BSD
On UNIX systems, the cookie will be typically located in /var/lib/rabbitmq/.erlang.cookie (used by the server) and $HOME/.erlang.cookie (used by CLI tools). Note that since the value of $HOME varies from user to user, it's necessary to place a copy of the cookie file for each user that will be using the CLI tools. This applies to both non-privileged users and root.

也就是说在Linux上，默认的cookies是在 `/var/lib/rabbitmq/.erlang.cookie`， 但是该rabbitmq的home目录是`root`，所以需要将默认路径的cookies替换到home目录下。替换之后即可。

# 关于Rabbitmq的web界面启动不了
提示如下：
```
Plugin configuration unchanged.

Applying plugin configuration to rabbit@izm5e0h94dt7do1kplgd15z... failed.

```

发现只要重启下rabbitmq就好了。。。



# 另外就是erlang的版本最好在20，在21貌似会出问题，当然针对rabbit3.6.15版本