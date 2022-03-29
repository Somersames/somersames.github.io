---
title: Springboot自定义@EnableXX注解
date: 2020-05-26 00:05:16
tags: [SpringBoot]
categories: [Java,SpringBoot]
---
在SpringBoot中，经常可以看到许多以 `@Enable` 开头的注解，例如：`@EnableAutoConfiguration`，`@EnableAsync`......，那么我们是否可以自己定义一个注解呢？

其实自定义注解最终都是利用到了 `ImportBeanDefinitionRegistrar` 这个类，通过手动的方式，将一个类注册成为 `Bean`，然后在进行一系列的操作，下面就来看下 `ImportBeanDefinitionRegistrar`
## ImportBeanDefinitionRegistrar
这个类的代码如下：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/how%20to%20define%20a%20annotion/ImportBeanDefinitionRegistrar.png)

可以看到这个类的结构很简单，就是一个方法，那么下面来看下这两个参数是什么意思。
### AnnotationMetadata
从字面的意思上可以看出来是一个注解的元数据，它的里面的的方法如下：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/how%20to%20define%20a%20annotion/AnnotationMetadata.png)
都是一些获取注解信息的方法，那么第二个参数呢

### BeanDefinitionRegistry

从字面的意思上可以看出来这个类是用于注册 `Bean` 的，其中最常见的就是 `registerBeanDefinition` 方法。它提供了两个参数 `String beanName, BeanDefinition beanDefinition`，而具体使用，则来看看demo。


## demo
假设现在有一个需求是要写一个切面，这个切面负责打印`controller` 的log。但是可能某些系统有自己的日志格式，不太需要这个切面AOP，所以希望可以增加一个开关，然后是按需引用。
下面就来开始写一个这样的demo，项目结构如下：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/how%20to%20define%20a%20annotion/project-tree.png)

在这里为什么会初始化两个文件夹，是因为 `@SpringBootApplication` 这个注解会默认将当前目录以及它的下级目录下的 `Bean` 注入到容器中，所以新建一个同级目录`anno` 就是为了不让 `@SpringBootApplication` 加载切面。

然后启动 `Application` 这个类，其中 `RestControllerTest` 就是一个很简单的 `Controller`，如下：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/how%20to%20define%20a%20annotion/RestControllerTest.png)

然后请求`http://localhost:8081/api/query/1`，可以看到在控制台没有任何的输出，那就说明自定义的切面还没有生效，此时在 `Application` 上添加我们的自定义注解，添加后代码如下：
```java
@SpringBootApplication
@EnableLog
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class);
    }
}
```

此时在此启动项目，你会发现已经已经在控制台有我们的日志log了。
实现的细节见下面几个类：

## 自定义注解
### @interface
首先，我们需要定义一个注解，其代码如下：
```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
@Import({LogRegister.class})
public @interface EnableLog {
}


```

在这里使用了 `@Import` 这个注解，由于不是本文的重点，因此不讲述其作用,（后面讲自动装配的原理时会讲解）然后这里有一个 `LogRegistrar` 类，这个类就是实现自定义注解的关键。

### LogRegister
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/how%20to%20define%20a%20annotion/LogRegister.png)

这个类的作用，就是让 `BeanDefinitionRegistry` 将我们的 `LogAop` 注册成一个Bean，只有当注册成一个 `Bean` 以后，该切面才会生效。

### LogAop
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/how%20to%20define%20a%20annotion/LogAop.png)

该类就是一个切面类，负责打印请求的耗时。自此，一个自定义注解就完成了，当第三方想接入该日志log的时候，就可以直接使用 `@EnableLog` 来开启。
> ps：注意调整`@Pointcut` 的切点，否则会切不到

这就是最简单的自定义注解了，至于里面涉及到的原理，后续可能会写一些文章来补充。

如果需要查看源代码的话，可以访问如下 `github` 地址：
> https://github.com/Somersames/spring-doc-sample/tree/master/anno-test