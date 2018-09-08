---
title: maven和json以及spring的一些问题
date: 2018-07-18 22:04:28
tags: [maven,spring]
categories: 工具
---
## maven和IDEA的一个问题
在 IDEA 中可以正常使用maven的一些命令来进行 clean 和 complime ，但是在使用IDEA的build功能时一直提示某些包找不到，解决办法：
执行 `mvn clean` 命令清除缓存，然后删除 `.idea` 这个文件夹中的文件

如果还是解决不了则可以直接换一个 maven ，最好的解决办法则是每一个项目，一个 maven。

## 阿里的Json包和对象之间的转换
今天有一个新的需求是将一个 Json 字符串转换成一个Json对象，此时可以调用
`JSON.parseObject( new TypeReference(XXX),json串)`来讲一个 Json 字符串转为一个对象


## spring中读取配置文件相关的问题
在 spring 中可以通过 `@Value` 这个注解来获取到配置文件中的一些配置，但是记住....不要再使用 `new` 关键字来再初始化