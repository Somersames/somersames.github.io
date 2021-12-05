---
title: Spring Cloud的Zuul相关总结
date: 2018-05-23 18:00:14
tags: [SpringCloud,web后端]
categories: [Java,SpringCloud]
---
Zuul是SpringCloud生态体系中的网关一环，首先简单配置如下：
开启注册中心并且配置yml文件，如下：
```yml
spring:
  application:
    name: somersames-erueka

server:
  port: 8081

eureka:
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://localhost:8081/eureka/
```

开启用户的微服务：
```yml
spring:
  application:
    name: somersames-user
server:
  port: 8082
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8081/eureka
```
编写一个测试的Controller：
```java
@RestController
@RequestMapping("testuser")
public class UserTestController {
    @GetMapping("/testzuul")
    public String testZUul(){
        return "测试Zuul";
    }
}
```

配置注册中心：
```yml

spring:
  application:
    name: somersames-zuul
eureka:
  client:
    service-url:
        defaultZone: http://localhost:8081/eureka
zuul:
  routes:
    user-route:
      url : http://localhost:8082  //用户微服务的地址
      path : /user/**          //映射的路径
server:
  port: 8083
```

设置Zuul启动类的注解：
```java
@SpringBootApplication
@EnableZuulProxy
public class ZuulConfigApplication {
    public static void main(String[] args) {
        SpringApplication.run(ZuulConfigApplication.class);
    }
}
```


测试：
使用微服务的SericiceId访问：
![](So.png)

使用Zuul的path访问：
![](su.png)