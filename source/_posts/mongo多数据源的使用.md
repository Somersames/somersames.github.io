---
title: mongo多数据源的使用
date: 2018-12-05 00:21:27
tags: [mongo]
---
## 简介
在开发的过程中，不可避免的会使用多数据源，但是相对于Mysql的多数据源，Mongo的多数据源配置还是比较容易的。

首先在`pom.xml`中引入`mongo`的驱动jar包以及`springboot`和`mongo`的一个jar包

## 原理
Mongo的多数据源无非是首先读取配置文件，生成`MongoProperties`，通过`MongoProperties`来生成一个`MongoTemplate`,最后通过`Repository`来操作Mongo。


而多数据源就是生成多了`MongoTemplate`，然后通过多个`MongoTemplate`所对应的`Repository`来操作Mongo数据库


## 代码

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-mongodb</artifactId>
            <version>${spring.version}</version>
            <exclusions>
                <exclusion>
                    <groupId>org.mongodb</groupId>
                    <artifactId>mongodb-driver</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
           <!-- Mongo相关-->
        <!-- https://mvnrepository.com/artifact/org.mongodb/mongo-java-driver -->
        <dependency>
            <groupId>org.mongodb</groupId>
            <artifactId>mongo-java-driver</artifactId>
            <version>3.8.0</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.mongodb/bson -->
        <dependency>
            <groupId>org.mongodb</groupId>
            <artifactId>bson</artifactId>
            <version>3.8.0</version>
        </dependency>
```