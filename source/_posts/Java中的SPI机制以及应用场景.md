---
title: Java中的SPI机制以及应用场景
date: 2021-08-04 01:14:19
tags: [Java]
categories: Java
---
SPI 全称是 Service Provider Interface，是 JDK1.5 新增的一个功能，允许不同的服务提供者去实现某个规定的接口，而且将具体的实现完全提供给使用方，允许使用方按需加载服务提供方的一些功能。

# 前言
提到 SPI，就不得不提下 API，以 dubbo 为例，服务提供方对外提供一系列 API，而使用方是不用关心服务提供方是如何实现具体的业务逻辑，只需要通过 RPC 调用远程服务即可。

![API](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/Java%E4%B8%AD%E7%9A%84SPI%E6%9C%BA%E5%88%B6%E4%BB%A5%E5%8F%8A%E5%BA%94%E7%94%A8%E5%9C%BA%E6%99%AF/api.png)

这样的好处就是 client 端不用关心服务端的具体逻辑，方便服务的水平扩展以及解耦。


# SPI
上面提到了 API 的相关知识，而 SPI 则是由服务方将具体实现提供给调用方，如何使用完全取决于调用方的具体业务逻辑，即调用方是可以拿到服务方的具体实现逻辑，然后决定是否使用，这有点像 Spring 的控制反转
![SPI](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/Java%E4%B8%AD%E7%9A%84SPI%E6%9C%BA%E5%88%B6%E4%BB%A5%E5%8F%8A%E5%BA%94%E7%94%A8%E5%9C%BA%E6%99%AF/spi.png)


实现该功能由如下两种方式：
1. 直接在代码中硬编码
2. 通过SPI来实现


## 第三方 jar
我们日常用 maven 将第三方 jar 导入到我们的项目中，大致思想和这个类似，都是将具体实现引入到 client，由 client 来决定使用的方式。
但是通过 maven 导入的有一个缺点，就是代码中存在硬编码，即需要 import 第三方包的类全路径，以后如果要替换具体实现，那么所有 import 了旧包的地方就需要全部修改一遍。

例如现在有一个接口如下：
```java
public interface Buy {
    Boolean buy(Integer var1);
}
```
而提供方则在自己的项目中实现了具体的逻辑：
```java
public class AliPayBuy implements Buy {
    public Boolean buy(Integer num) {
        System.out.println("buy" + num + "with AliPayBuy");
        return true;
    }
}
```
下面就来看看 maven 和 SPI 是如何具体实现的。

首先项目的整体结构如下，然后分别 deploy 到自己的私服中
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/Java%E4%B8%AD%E7%9A%84SPI%E6%9C%BA%E5%88%B6%E4%BB%A5%E5%8F%8A%E5%BA%94%E7%94%A8%E5%9C%BA%E6%99%AF/demo.png)

## 普通使用
如果不想通过 SPI 来调用，那么直接在项目中 new 一个也是可以的。
```java
public class SimpleTest {
    public static void main(String[] args) {
        Buy buy = new AliPayBuy();
        System.out.println(buy.buy(1));
    }
}
```
这种硬编码会导致后续完全没有扩展性，以后如果需要将支付改为其他方式，那么所有涉及到 AliPayBuy 的地方全部都得替换。

## SPI
而 JDK 的作者为了解决这个问题，引入了 SPI 机制，具体来说就是定义了一个文件夹「META-INF/services」，调用方规定一个接口，
提供方则在自己的项目中实现具体的逻辑，然后在自己的项目中将具体实现放置在「META-INF/services」即可。

```java
public class SPITest {
    public static void main(String[] args) {
        ServiceLoader<Buy> shouts = ServiceLoader.load(Buy.class);
        for (Buy s : shouts) {
            System.out.println(s.buy(1));
        }
    }
}
```
而采用 SPI 机制，可以避免在代码中直接引入第三方 jar，`ServiceLoader.load` 加载的正是之前 deploy 进私服的 alipay jar 包。

### META-INF/services
JDK 中规定，只有在这个文件夹中的 SPI 才会被加载，因为 JDK 已经将该路径硬编码到代码中了，而文件的命令也很有规范。
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/Java%E4%B8%AD%E7%9A%84SPI%E6%9C%BA%E5%88%B6%E4%BB%A5%E5%8F%8A%E5%BA%94%E7%94%A8%E5%9C%BA%E6%99%AF/maven_nexus.png)

文件名称必须是接口的全路径名称（大小写也必须一致）
而文件里面的内容就是实现该接口的类的全路径名称，例如 `xyz.somersames.Buy` 这个里面的内容就是
```java
xyz.somersames.AliPayBuy
```

# 双亲委派机制
通过 SPI 机制加载的类是会破坏双亲委派机制的，因为按照双亲委派的机制，当 classLoader 加载某一个类的时候是一层一层往上递增的，然后再逐级往下，但是 SPI 机制是通过 thread.contextClassLoader 直接加载了具体的实现类。
虽然说原生的 classLoader 是按照双亲委派机制在加载类，但是 `thread.contextClassLoader` 这里由于极大的灵活性可能会导致被用户的自定义 classLoader 覆盖，而如果用户自定义的 classLoader 不按照规范来，那么就直接破坏了双亲委派机制了

其二，因为 JDK 的 SPI 接口一般是位于 rt.jar 中，按照双亲委派机制，应该由 BootstrapClassLoader 加载，但是其实现类却是位于 classPath，由 AppClassLoader 加载，所以这也算是破坏的一种