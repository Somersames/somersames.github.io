---
title: 使用lambda来优化责任链模式
toc: true
date: 2021-12-12 22:38:49
tags: [Java,设计模式]
categories: [Java,设计模式]
---
责任链模式是设计模式的一种，可以为调用的对象进行一个链式处理，这种模式在 Java 的一些第三方库中经常见到。

而在业务开发中，这种需求也是很常见的，如果用好这个设计模式，对于代码的扩展性和可维护性都是非常有帮助的，例如常见的下单流程，就可以用责任链模式处理。

而在第三方库中，像 tomcat 的过滤器就是使用责任链模式进行处理

# Tomcat 中的使用

在 tomcat 中，过滤器的实现就完全是责任链模式的使用了

![image-20211213233708542](	https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/%E4%BD%BF%E7%94%A8lambda%E6%9D%A5%E4%BC%98%E5%8C%96%E8%B4%A3%E4%BB%BB%E9%93%BE%E6%A8%A1%E5%BC%8F/image-20211213233708542.png)

首先 tomcat 定义了一个 Filter 类，这个类有三个方法，其中 init 和 destroy 为 default 方法，因此不要求子类必须实现，而 doFilter 则是一个必须实现的方法，正是这个方法实现了链式调用。

如果之前有使用过 servlet 的同学应该知道，如果需要使用过滤器，则需要在要在 web.xml 里面配置所需要加载的 Filter，tomcat 会将所配置的过滤器加载到一个数组里面去，最后在调用 doFilter 的时候，选择 chain 进行传递

# Java 中常见的用法

Java 中的最常见用法有两种

* 一种是用一个 List 将所有的实现类全部添加到一个集合里面
* 一种就是通过 List 的 iterator 进行迭代调用

# Lambda表达式

JDK8 中一个大升级就是提供了 lambda 表达式，而这个特性又为责任链模式的使用提供了一个新的实现方式

## Function

函数式接口是 JDK8 推出的一个升级点，不同于之前的接口，函数式接口有且只能有一个抽象方法，并且接口被 `FunctionalInterface`修饰，配合 Stream 类，可以方便快速的实现各种操作

![image-20211216005518041](	https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/%E4%BD%BF%E7%94%A8lambda%E6%9D%A5%E4%BC%98%E5%8C%96%E8%B4%A3%E4%BB%BB%E9%93%BE%E6%A8%A1%E5%BC%8F/image-20211216005518041.png)

该接口可以将一个范型为 T 的入参，转化为 R 类型的返回值，如果单独来看仅仅就是一个普通的接口，在 JDK8 之前的版本也可以实现类似的功能。

但是 JDK8 中配合 Stream 就可以实现非常大的提升。

# Stream

Stream 是 JDK8 提供的一个非常强大的工具类，能将集合转为为流，通过一系列的操作，将流转化为某种结果

## reduce

reduce 是 Stream 类中的一个方法，可以将流进行合并，并且产生一个新的值，这个值可以是一个 函数式接口，也可以是一个具体的某个值，下面是其三个方法

![image-20211218114550489](	https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/%E4%BD%BF%E7%94%A8lambda%E6%9D%A5%E4%BC%98%E5%8C%96%E8%B4%A3%E4%BB%BB%E9%93%BE%E6%A8%A1%E5%BC%8F/image-20211218114550489.png)

其中参数个数依次变多，不过今天主要是来研究前两个，利用这两个函数可以实现一个非常优雅的责任链吊用

### 数组求和Demo

![image-20211218114949668](	https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/%E4%BD%BF%E7%94%A8lambda%E6%9D%A5%E4%BC%98%E5%8C%96%E8%B4%A3%E4%BB%BB%E9%93%BE%E6%A8%A1%E5%BC%8F/image-20211218114949668.png)

这是一个数组求和的demo，分别使用了 reduce 的前两种用法，不过第一个返回的是 Optional，所以需要 get 操作。

而第二个方法由于第一个参数已经指定了返回的类型，因此无需再操作get，并且第二个方法的第一个参数其实也是一个初始值，可以将其与最后的结果累加。



在这里如果继续思考一下，如果 reduce 里面是一个函数，并且第一个函数的值作为第二个函数的入参，那么是不是就可以实现责任链了。

也就是说如果返回值是 Function，然后在 Function 中实现一个默认的方法，这个方法的返回值也是一个 Function，那么是不是就可以达到这个目标了。





# 定义接口

因为 reduce 的入参需要是 Function 函数，因此在这里定义一个函数

![image-20211218140454129](	https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/%E4%BD%BF%E7%94%A8lambda%E6%9D%A5%E4%BC%98%E5%8C%96%E8%B4%A3%E4%BB%BB%E9%93%BE%E6%A8%A1%E5%BC%8F/image-20211218140454129.png)

其中 andThen 的思想就是将上一个 Function 的结果作为下一个 Function 的入参，然后再递归进行调用

```java
after.apply(apply(t))
```

如果对算法比较了解的同学，可以看到这个方法就是一个递归调用，直至调用到达最后一个 Function。

那么接下来就定义一些 Processor

## 实现

首先定一个查询商品的 Processor，该 Procrssor 通过商品的 ID 来获取商品的详细信息，包括库存数以及价格等等。

然后将商品的详细信息带入到优惠卷查询 Processor 里面去，最后判断该商品有哪些优惠卷

![image-20211218142652500](	https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/%E4%BD%BF%E7%94%A8lambda%E6%9D%A5%E4%BC%98%E5%8C%96%E8%B4%A3%E4%BB%BB%E9%93%BE%E6%A8%A1%E5%BC%8F/image-20211218142652500.png)

然后返回的 Goods 可以作为优惠卷查询 Processor 的入参，获取用户所包含的优惠券

![image-20211218142856242](	https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/%E4%BD%BF%E7%94%A8lambda%E6%9D%A5%E4%BC%98%E5%8C%96%E8%B4%A3%E4%BB%BB%E9%93%BE%E6%A8%A1%E5%BC%8F/image-20211218142856242.png)



## 链式调用

首先将这两个 Processor 加入到一个集合里面去，方便转成 Stream，然后通过 reduce 调用 andThen 函数

![image-20211218143038040](	https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/%E4%BD%BF%E7%94%A8lambda%E6%9D%A5%E4%BC%98%E5%8C%96%E8%B4%A3%E4%BB%BB%E9%93%BE%E6%A8%A1%E5%BC%8F/image-20211218143038040.png)

 最终的结果是，如果 apply 的入参是 d1，那么返回的结果如下：

```java
OrderInfo{goods=Goods{id='d1', remain=10, price=1}, coupon=Coupon{idList=null, discount=null}}

```

而如果入参是 "1"，那么得到的结果就是：

```java
OrderInfo{goods=Goods{id='1', remain=10, price=1}, coupon=Coupon{idList=[1], discount=10}}

```

于是通过 JDK8 的 lambda 就可以实现一个简单的责任链了

## Context 传递

也许有人说，这样的话，如果需要修改顺序，那入参的地方都要改，非常的不友好，别慌，还有 BiFunction

如果需要在一个链式调用中，通过 Context 来保存调用的上下文，那么可以这样实现

![image-20211218152941016](	https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/%E4%BD%BF%E7%94%A8lambda%E6%9D%A5%E4%BC%98%E5%8C%96%E8%B4%A3%E4%BB%BB%E9%93%BE%E6%A8%A1%E5%BC%8F/image-20211218152941016.png)

每一个实现 Processor 的类第一个参数都会是 OrderContext，而在调用的地方其实也没什么区别，就是多加了一个入参，所以非常的方便。

# 配合 Spring 使用

如果配合 Spring 来使用，那么还有一个更加便捷的方法，每一个 Processor 都实现 Ordered 接口，然后通过 @Resouces 直接注入进来，非常的方便

# 最后

其实在做业务的时候，如果可以抽空用 JDK 的新特性优化下之前的老代码，是一个非常不错的提升机会，既可以增加自己对业务的理解程度，又可以将所学用于业务开发，可以在日常的开发中尝试一下

本文的[demo地址](https://github.com/Somersames/article-demo/tree/master/responsibility-demo)

