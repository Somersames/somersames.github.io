---
title: SpringCloud初步配置之起步
date: 2018-04-18 23:58:34
tags: [web后端,SpringCloud]
categories: [Java,SpringCloud]
---
spring cloud是一个微服务治理框架，主要解决的是以前单应用过于臃肿的一些难点，在这里记录下初次使用springcloud的一些过程

## 创建一个springboot应用
在这里使用的是IDEA的`spring Initializr`创建的，创建好了之后将`application.properties`改成`application.yml`,其项目结构图如下：
![](项目结构图.PNG)

在这里需要注意的是由于在这里的Test文件中含有一个`@RunWith`注解，所以需要在pom.xml中引入springboot的test包，否则项目可能启动不起来
## 引入spring cloud包
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.4.5.RELEASE</version>
</parent>
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Camden.SR7</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
<dependencies>
    <dependency>
        <groupId></groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
    <dependency>
        <groupId></groupId>
        <artifactId>spring-cloud-starter-eureka</artifactId>
    </dependency>
</dependencies>
```

这是官网的以一个配置，但是在引入本地的时候还是需要稍微做点修改。
修改之后的：
```xml
<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.4.5.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-config</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-eureka-server</artifactId>
		</dependency>
	</dependencies>
	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>Camden.SR7</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

```

在这里添加了`>spring-cloud-starter-eureka-server`,而且在这里引入的jar都不需要添加版本号。

## 修改配置文件

```yml
spring:
  application:
    name: stock-service

server:
  port: 8083
eureka:
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://localhost:8084/eureka/
```

最后便可以启动项目了。

## 结果
运行出来的结果如下：
![](eureka.PNG)
