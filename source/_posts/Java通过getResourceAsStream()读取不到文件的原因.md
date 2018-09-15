---
title: Java通过getResourceAsStream()读取不到文件的原因
date: 2018-09-10 23:45:30
tags: [Java]
categories: Java
---

首先出现这个原因的时候，需要弄清楚工程目录和编译目录。
## 工程目录
以IDEA为例，在IDEA里面，我们写代码的地方就是一个工程目录，常见的例如`src`下面的各种java文件，这种目录就可以称之为一个工程目录，例如如下所示：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E5%B7%A5%E7%A8%8B%E7%9B%AE%E5%BD%95.png)

工程目录主要存放的是一些配置文件或者一些java文件之类的，而经jvm编译之后的目录便是编译目录了

## 编译目录
编译目录主要用于存放java编译后的class文件，也就是我们运行的文件。众所周知，java是一种跨平台语言，所以jvm实际运行的是java变异之后的class文件。
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E7%BC%96%E8%AF%91%E7%9B%AE%E5%BD%95.png)

当知道了这个之后便会理解为什么会通过`getResourceAsStream()`读不到文件了。

## getResourceAsStream()

翻开Java的官方文档，可以看到`getResourceAsStream()`是ClassLoader的一个方法，
```java
static InputStream	getSystemResourceAsStream(String name)
Open for reading, a resource of the specified name from the search path used to load classes.
```
那么一般获取Java配置文件的代码如下：
```java
InputStream inputStream = DemoTest.class.getClassLoader().getResourceAsStream();
```
这个时候程序运行起来了，那么她就不会去工程目录下寻找配置文件了，例如在如下工程里面运行如下代码：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E8%BF%90%E8%A1%8C%E6%B5%8B%E8%AF%951.png)
```java
public class DemoTest {
    public static void main(String[] args) {
        InputStream inputStream = DemoTest.class.getClassLoader().getResourceAsStream("mybatis-config.xml");
        if (inputStream == null){
            System.out.println("获取异常");
        }else{
            System.out.println("获取到了文件");
        }
    }
}
程序打印如下：
获取异常
```

这个时候就会出现`getResourceAsStream`获取不到文件了，那么假如将`mybatis-config.xml`复制到target的目录下面去呢?
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E8%BF%90%E8%A1%8C%E6%B5%8B%E8%AF%952.png)

再次运行该代码，控制台打印：获取到了文件。

所以遇到了这个情况的话一般就是工程目录和编译目录缺少文件了。


## 如何找到ClassLoader的获取文件目录呢

只需在`Resource`类下面debug`getResourceAsStream`，然后打开loader即可，找出`domain`属性就可以看到了。
```java
public static InputStream getResourceAsStream(ClassLoader loader, String resource) throws IOException {
    InputStream in = classLoaderWrapper.getResourceAsStream(resource, loader);
    if (in == null) {
      throw new IOException("Could not find resource " + resource);
    }
    return in;
  }
```

![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/mybatis.png)

可以看到那个就是一个获取的编译目录。