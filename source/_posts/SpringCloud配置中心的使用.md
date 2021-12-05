---
title: SpringCloud配置中心的使用
date: 2018-06-11 22:29:16
tags: [SpringCloud,web后端]
categories: [Java,SpringCloud]
---

在实际的开发过程中，很可能会涉及到很多的开发环境，常见的例如 dev , product 等，在使用SpringCloud的时候可以通过配置中心微服务，结合 Git 管理工具实现配置的集中式管理。

## 配置中心
config配置中心的yml文件：
```yml
spring:
  application:
    name: sc-config
  cloud:
    config:
      server:
        git:
          uri: ${GitAddress:https://gitee.com/somersames/spring-cloud-demo-config}
          search-paths: ${GitPath:dev}
eureka:
  client:
    service-url:
      defaultZone: http://${EurekaHost:localhost}:${EurekaPort:8081}/eureka/
server:
  port: 8888
```
这里的 `${A:B}` 配置表示的是如果 A 获取不到就取 B 的值。
`search-paths` 则是代表从哪个文件夹中获取配置文件，可以使用 `,` 来进行分割。
然后在启动类中添加如下几个注解，一个配置中心的微服务便开启了。
```java
@SpringBootApplication
@EnableConfigServer
@EnableAutoConfiguration
@EnableEurekaClient
public class ConfogApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfogApplication.class);
    }
}
```

## 获取配置中心的配置

当其他几个微服务需要通过配置中心获取配置文件的时候，需要添加一些配置文件才可以。
```yml
spring:
  profiles:
    active: master
  application:
    name: sc-auth
  cloud:
    config:
      discovery:
        service-id: sc-config #注册中心的ServiceId
        enabled: true
      label: master #表示的是分支
      profile: config #表示的是一个标签
```
那么这个微服务在拉取配置文件的时候就会拉取 `sc-auth-config.yml` 的配置
