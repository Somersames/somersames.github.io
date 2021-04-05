---
title: Springboot的maven间接依赖
date: 2020-05-19 00:21:53
tags: [maven]
categories: Springboot
---
在项目中经常使用 `maven` 来管理项目，但是有时候对于 `maven` 的细节还是了解的不是很清楚，因此今天复习下。

## maven项目
首先开始建立一个最简单的 `maven` 项目，其配置如下图：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/maven-project/maven-project.png)
 
可以看到最上面一行是 `xml` 的文件描述符，然后再是 `project`，在这里引入 xsd 文件。
> XSD(XML Schemas Definition)XML Schema，描述了 xml 文档的结构，用于判断其是否符合 `xml` 的格式要求

然后下面就是 `groupId`，通常是公司的域名，`artifactId` 通常指的是项目名称。

## Springboot项目
按照官方的指导，在项目中首先引用 `spring-boot-starter-parent`，修改后的 `pom.xml` 如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>maven-test</artifactId>
    <version>1.0-SNAPSHOT</version>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.2.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
    <finalName>maven-test</finalName>
</build>
</project>
```


当准备在启动类上加 `@SpringBootApplication` 注解的时候，此时 IDEA 会提示找不到这个注解。这是正常的，因为 `parent` 只是把这个项目的配置和依赖信息统一化了，使得 `子pom` 就不用关心版本问题，例如在项目中引入`spring-boot-starter-web`，当配置了 `parent` 之后，只需要在 `子pom` 中如下配置：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

那么它的配置信息就会自动的从 `parent` 中读取。例如刚刚的`spring-boot-starter-web`信息，它的版本信息如下：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/maven-project/maven-web-version.png)
> tips:使用命令`mvn -Dverbose dependency:tree`就可以像这样打印 jar 的依赖

那么 springboot 又是怎样来自动识别版本号的呢，此时就就涉及到了`spring-boot-dependencies`


## spring-boot-dependencies
`spring-boot-dependencies` 是 `spring-boot-starter-parent` 的一个 parent，可以在 `spring-boot-starter-parent` 的 `pom` 文件中看到。
打开 `spring-boot-dependencies` 文件，你会发现它里面几乎全部都是一些配置信息，而刚刚的`spring-boot-dependencies` 版本号就是来自于此。
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/maven-project/depence-maven.png)

到目前为止，可以基本理清 `springboot` 的依赖关系了。
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/maven-project/pom-relation.png)


## 打包
在工程中，随便写一个 `controller`，然后执行`mvn package`，此时会在 `target` 目录下出现一个 jar 包，然后运行 jar 包，正常启动OK。

## 替换parent
既然 `spring-boot-starter-parent` 是依赖于 `spring-boot-dependencies`的，那么可不可以直接将`parent` 设置为`spring-boot-dependencies`呢，修改 pom 文件如下:
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/maven-project/replace-parent.png)

然后执行`mvn package`，执行的时候是成功的，但是当你用 `java -jar maven-test.jar` 的时候，你会发现提示如下：
```shell
target java -jar maven-test.jar
maven-test.jar中没有主清单属性
```

### 原因
首先分析下两个的 `maven` log。

#### **spring-boot-starter-parent作为parent**

![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/maven-project/parent-log.png)

#### **spring-boot-dependencies作为parent**

![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/maven-project/depence-log.png)

可以看到第二次的打包插件是 `maven-jar-plugin`，也就是说 springboot 的项目一些资源并没有打包进来，查看 `spring-boot-maven-plugin` 插件，发现它是来自于 `spring-boot-starter-parent` 里面的，但是在文章的开头部分，是已经手动的将其引入到了 pom 文件，那么修改 parent 以后未执行的话，最有可能就是版本号的缺失导致的，于是修改pom：
```xml
 <plugins>
    <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
        <version>2.1.11.RELEASE</version>
    </plugin>
</plugins>
<finalName>maven-test</finalName>
```
然后运行`mvn package`：
```java
[INFO] --- spring-boot-maven-plugin:2.1.11.RELEASE:repackage (repackage) @ maven-test ---
[INFO] Replacing main artifact with repackaged archive
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
```
你会发现此时打包出来的 jar 文件已经可以运行了。

## 进阶
那么假设项目已经有了自己的 `parent`，如果还想用 `spring-boot-dependencies` 来进行统一的一个全局版本控制，那么有如下的解决办法

在自己的`parent`中设置parent为 `spring-boot-starter-parent`，那么根据 maven 的继承属性，所有的 `子pom` 也就顺带继承了 `spring-boot-starter-parent`