---
title: ClassNotFound 和 NoClassDefFoundError 的区别
toc: true
date: 2022-04-07 00:33:22
tags: [maven]
categories: 工具
---
在 Java 的开发中，这两个可能是让人比较头痛的异常了，因为出现这个异常，又得一大堆 jar 包冲突需要解决。


# ClassNotFound
按照 oracle 官方的描述：[Class ClassNotFoundException](https://docs.oracle.com/javase/8/docs/api/java/lang/ClassNotFoundException.html)
> Thrown when an application tries to load in a class through its string name using:
    The forName method in class Class.
    The findSystemClass method in class ClassLoader .
    The loadClass method in class ClassLoader.

也就是当 JVM 尝试着去加载某个类的时候，会发现 classPath 下面却没有这个类，那么就会直接报错，在日常开发中出现的常见的常见原因是，两处引用了不同版本的第三方包（maven）。

maven 中采用的是就近原则，并且 maven 中有一个规定就是：groupId、artifactId 相同的 jar 包，只会选择一个版本进行引入，而引入的版本如下：
> 路径最短者优先原则，路径相同先声明明优先原则

![image-20220421230816084](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/difference%20between%20ClassNotFound%20and%20NoClassDefFoundError/image-20220421230816084.png)

例如上图中的 1.4 版本，就不会被 maven 所选择，这是因为 spi_client 的路径是相同的，而 1.5 版本则更近，所以 maven 就会选择 1.5 的版本加载到 classPath。

还有一种就是这个情况：

![image-20220421233355690](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/difference%20between%20ClassNotFound%20and%20NoClassDefFoundError/image-20220421233355690.png)

此时 maven 就会选择 1.0 版本的 D，因为 1.0 版本的到达项目的更近。


此时，如果项目中使用了其他版本的 jar 包，就会出现 ClassNotFount 异常。

# NoClassDefFoundError

这个就比较特殊了，具体的含义就是项目编译、加载通过，但是在使用的时候却发现突然没有了 class 文件。

![image-20220421234848959](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/difference%20between%20ClassNotFound%20and%20NoClassDefFoundError/image-20220421234848959.png)

上图是我们最近遇到该异常的一个依赖关系图

service会通过 bean 容器获取到对应的 worker 然后调用对应的方法，但是由于公司的插件检测到了版本冲突，所以手动在 service 模块中去除了 worker 中的 api2.0

```xml
<exclusions>
  <exclusion>
    <groupId>XXX</groupId>
    <artifactId>api</artifactId>
  </exclusion>
</exclusions>
```

因此 service 中实际上的  api 版本是 1.0 的，而一旦项目运行起来，在 service 中存在的是 1.0 版本的 api，如果 worker 中有方法以来 api2.0，那么就会直接报错。


# 最后

正常情况下，message都会给出明显的提示，静态代码报错，直接看 Caused By，非静态的一般会提示具体的类名



## 解决方法

1. 将冲突的版本的 jar 版本号统一
2. 如果冲突的 jar 是由于 maven 的就近原则引入进来的，则直接排除掉即可
3. 使用阿里的 pandora 自定义插件处理

