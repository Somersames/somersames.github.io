---
title: 记一次springboot-mybatis找不到dao层的错误
date: 2018-03-05 20:23:02
tags: [mybatis]
categories: [Java,Spring]
---
写了三个多月的Python，今天再写一个springboot-mybatis的项目的时候好多东西都忘记了，尤其是在今天下午遇到了一个关于mybatis的错误：
```
Description:

Field peopleDao in com.example.serviceImpl.PeopleServiceImpl required a bean of type 'com.example.dao.PeopleDao' that could not be found.


Action:

Consider defining a bean of type 'com.example.dao.PeopleDao' in your configuration
```
很明显就是dao层无法被扫描到，在一下午的尝试中，首先检查了包结构，如下:
com
---example
------controller
------------XXX.java
------dao
------------XXX.java
--------***
------Application.java

也就是说这个包结构是完全符合springboot的规范的，也就表示在启动类上面的注解是完全可以扫描到dao包下面的那个接口的。所以注解不存在问题。

然后尝试了第二种方法。修改启动类的扫描结构：
```
@SpringBootApplication
@ComponentScan(basePackages = {com.example.dao})
public class IpApplication {
	public static void main(String[] args) {
		SpringApplication.run(IpApplication.class, args);
	}
}
```
修改之后确实不会报错了但是有一个问题就是除了这个dao下面的包可以被扫描之后其他的例如controller包都无法被扫描进来。所以也是无法根本解决问题，

最后无意间注意到了在Springboot的启动日志中出现了一句话：**NO Mybatis mapperXXX**后面的具体忘了，也就是说在这里根本就没有扫描到mapper，后来发现是查了一个jar包
```
		<dependency>
			<groupId>org.mybatis.spring.boot</groupId>
			<artifactId>mybatis-spring-boot-starter</artifactId>
			<version>1.3.1</version>
		</dependency>
```
也就是这个jar包导致了那个mapper一直找不到，不过具体原因待后面时间再补充