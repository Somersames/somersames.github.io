---
title: 为什么Spring官方推荐通过构造器注入
date: 2019-07-22 00:16:15
tags: [Spring]
categories: Spring
---
## 一

我们在使用Spring的时候，最方便的就是它的IOC（控制反转），也就是所有的Bean都交由

Spring进行管理，那么我们在看网上的文章或者自己在写代码的时候经常会像这样写：

```java
@Service
public class TestService {

   @Autowired
   TestDao TestDao;

   public void getInfo(){
   }
}

@Service
public class TestService {

   private final TestDao TestDao;

   @Autowired
   public TestService(TestDao TestDao){
     this.TestDao = TestDao;
   }

   public void getInfo(){
   }

}
```

但是貌似许多人都会使用第一种方式，因为这样简单方便，如果是采用第二种的话，每一次新增加一个bean，都需要在构造器的入参上面加一个参数，就会显得有点麻烦。

但是如果采用第一个写法，就会在IDEA里面出现一个提示：

>  spring官方建议通过构造器的方式进行注入

这又是为什么呢？

[Spring官方对于Setter注入和构造器注入的看法](https://spring.io/blog/2007/07/11/setter-injection-versus-constructor-injection-and-the-use-of-required/)



## 二

在springboot里面，最常用的注入方式有两种：一种是构造器注入，一种是field注入

在上一部分的代码里面，第一个是 Field 注入，第二个是构造器注入，既然这两种方式Spring都支持，那么到底这两种有什么区别呢？

### Field注入

这边新建两个测试类：

```java
@Service
public class A {

    public void sayA(){
        System.out.println("A");
    }
}

@Service
public class B {
    @Autowired
    A a;

    public B(){
        a.sayA();
    }
}
```

此时你启动Spring的话，就会出现一个空指针的异常，如果需要避免的话，则必须是类 `A` 先进行初始化，然后再初始化 `B` 。（当然Spring官方也提供了很多方式来控制 Bean 的初始化顺序，但是和本篇文章无关）



### 构造器注入

```java
@Service
public class B {
   
    private final A a;

    @Autowired
    public B(A a){
        a.sayA();
        this.a = a;
    }
}
```

此时Spring的项目就会正常的启动，那么为什么同样的代码，一个通过构造器注入，一个通过Field注入，两者的结果相差这么大呢？

## 三

官方之所以现在推荐使用构造器注入，是因为通过构造器注入是可以防止 `空指针异常`，同时可以确保的是被引用的 `Bean` ，它的引用是不可以变的，所以这可能是Spring官方团队的一些权衡点吧
当然早期的时候，Spring曾推荐过使用Setter注入，不够现在可能Spring可能觉得构造器注入比较好。