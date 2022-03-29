---
title: 优雅的dubbo服务端校验
toc: true
date: 2022-03-07 00:49:15
tags: [dubbo]
categories: dubbo
---
在业务的开发过程中，肯定会有一些数据字段的校验，例如手机号格式，用户名格式等等。

如果这个放到每一个具体的接口去判断的话，首先是和业务代码耦合，每一个接口的实现方都需要在代码中判断一系列的校验，而且后续如果需求产生变更，那么每一个在业务中进行判断的方法都需要改变，非常的耗时且不优雅

# 解决方案
目前无论是 spring全家桶 还是 dubbo，通用的做法就是通过 Hibernate Validator 来进行入参的校验，如果是配合 spring 使用的话，那么可以直接引入如下的 pom 文件：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
    <version>${spring-boot.version}</version>
</dependency>
```

本文主要是以 dubbo 为例子，其实如果用的是 spring 配合 HTTP 请求，实现上大致是差不多的


# 服务端
## 加入注解校验

![image-20220326231520822](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/how-to-validator-dubbo-server-in-server/image-20220326231520822.png)
> 更加优雅的做法是通过配置文件来控制提示信息

然后在开启服务端校验，虽然服务端和客户端均可以配置，如果这个因为业务规则是必须做校验，那么最好还是服务端开启。

因为你不知道客户端会靠不靠谱

![image-20220326231605154](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/how-to-validator-dubbo-server-in-server/image-20220326231605154.png)

然后定义一个方法，如下：

![image-20220327003238489](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/how-to-validator-dubbo-server-in-server/image-20220327003238489.png)

当我们开启参数注解以后，正常情况下，如果客户端用在户名不传递的情况下是无法得到正确的结果，那么开启一个客户端来调用试一下：

## 客户端

![image-20220327003351396](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/how-to-validator-dubbo-server-in-server/image-20220327003351396.png)

调用后的日志返回如下：

![image-20220327003508073](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/how-to-validator-dubbo-server-in-server/image-20220327003508073.png)

可以看到，客户端直接报错了，显然这种返回并不是非常的友好，如果参数不合法就直接提示报错，那么消费端可能还需要解析报错信息来进行处理，例如是否发送告警等等。

所以一个统一的返回是非常有必要的，不仅仅是可以优化业务的处理流程，而且也会提高系统的性能

> Java 中抛出异常，需fillInStackTrace信息，因此会造成一定的性能损耗



# Dubbo 文档

在 dubbo 文档中，有专门的一个章节来讲述 参数校验 [参数校验](https://dubbo.apache.org/zh/docs/v2.7/user/examples/parameter-validation/)

按照文档的配置，写好服务端和客户端以后，会发现虽然拦住了，但是 consumer 端却直接报错了，如下
> dubbo 服务端的版本是 2.7.4.1

![image-20220328001921744](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/how-to-validator-dubbo-server-in-server/image-20220328001921744.png)

>  这个报错会出现在低版本的 dubbo，如果版本较高，则不会有此类报错，主要原因还是 hessian 序列化的原因，但是因为不涉及到本文内容，如果不感兴趣可以忽略此节，而且网上的解释也比较多

看起来是无法序列化一些东西，于是在 issue 里面搜索了下相关问题，发现还真有人遇到了[Fix When using hibernate-validator, class 'org.hibernate.validator.internal.metadata.descriptor.ConstraintDescriptorImpl' is wrong with hessian deserialization](https://github.com/apache/dubbo/pull/1708)

但是呢这个 PR 并没有被合并到 master，原因在这里：

![image-20220328002210130](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/how-to-validator-dubbo-server-in-server/image-20220328002210130.png)

意思就是 dobbo 的开发者并不建议通过一个自定义的异常来处理验证失败的消息，然后提交 PR 的人问了下有不有更好的解决办法，dubbo 的维护人员就建议可以自定义一个 Filter 来处理此类问题。

### 疑问？

我的服务端如果用的是新版本（2.7.8）已经不会报这个序列化的错误了，是不是已经有人修复了呢？

于是去 github 上的 issue 上搜了下，还真发现了一个合并记录，这个老哥在 ValidationFilter 中新增一个 catch，而捕获的异常正好是 ValidationException，那是不是正好是这个改动，正好修复了上述的序列化问题呢？

[avoid of serialization exception for javax.validation.ConstraintViolationException](https://github.com/apache/dubbo/pull/5672)

![image-20220327011341956](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/how-to-validator-dubbo-server-in-server/image-20220327011341956.png)

这几行的改动是修复这个 bug： [Provider参数验证时，javax.validation.ConstraintViolationException序列化异常](https://github.com/apache/dubbo/issues/5432)

![image-20220329011534492]
而 ConstraintViolationException 和 ValidationException 的关系图如下：
(https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/how-to-validator-dubbo-server-in-server/image-20220329011534492.png)



可以看到 ConstraintViolationException 正好是 ValidationException 的一个子类，所以这个异常恰好被捕获而且被转换为 ValidationException，也因此避免了序列化异常了。

好了，到这里也就基本介绍了 dubbo 配合 validation 使用的现状
1. 低版本序列化异常
2. 高版本客户端直接报错了

那么如何进行优化呢？

## 自定义 Filter

既然作者是建议自定义一个 Filter，那么就参照 dubbo 维护者的建议来处理：

Dubbo 的自定义的 Filter 是通过 SPI 机制来实现的，需要在 resource 目录下新建一个文件，项目结构图如下：

![image-20220329225315825](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/how-to-validator-dubbo-server-in-server/image-20220329225315825.png)

## 自定义Filter：

其中主要的功能就是重写 invoke 方法

![image-20220329233517003](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/how-to-validator-dubbo-server-in-server/image-20220329233517003.png)

然后在配置中将其开启即可：

![image-20220329234153374](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/how-to-validator-dubbo-server-in-server/image-20220329234153374.png)



这样返回的信息里面就会是我们自定义的 Response 了

![image-20220329233439467](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/how-to-validator-dubbo-server-in-server/image-20220329233439467.png)



# 后续

这样通过一个 Filter 就可以让 consumer 得到真实的返回值，从而方便做业务的区分，当然也可以自定义一些自定义的注解来配合做更加复杂的处理了，不过这也是后话了。


# 参考
1. [Dubbo服务如何优雅的校验参数](https://mp.weixin.qq.com/s/0wtAGo12OtziVYt7YgGDlQ)