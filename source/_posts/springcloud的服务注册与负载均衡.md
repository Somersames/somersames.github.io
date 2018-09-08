---
title: springcloud的服务注册与负载均衡
date: 2018-04-20 23:33:00
tags: [web后端,springcloud]
categories: Spring Cloud
---
在springcloud中，服务注册和负载均衡分别是eureka和ribbon。其中eureka是一个服务注册组件，ribbon则是一个负载均衡组件。如果开启了hystrix之后ribbon就默认已经开启了。


## Eureka
在使用Eureka的时候服务端开启之后直接在客户端的application.yml中加入如下代码即可:
```yml
eureka:
  client:
    serviceUrl:
          defaultZone: http://localhost:8083/eureka/
```
其中defaultZone代表的是服务端的地址，开启服务之后在eureka的服务面板便会看到这个注册信息。如果需要在这个客户端中加入一些元数据的话可以加入如下配置：
```yml
instance:
  prefer-ip-address: true
    metadata-map:
      mydata: testydata
```
其中`metedata-map`下面就代表的是自己定义的元数据，这个会在其他客户端请求查看微服务信息的时候一同显示出来。



## ribbon
负载均衡组件，可以将请求均匀的分布在相同的微服务实例上，需要几个微服务都是同一个相同名称，端口可以不同。
新建两个客户端,第一个客户端端口号是8085
```yml
spring:
  application:
    name: eurekamovie
eureka:
  client:
    serviceUrl:
          defaultZone: http://localhost:8083/eureka/
server:
  port: 8085
```

第二个客户端端口是8087
```yml
spring:
  application:
    name: eurekamovie
eureka:
  client:
    serviceUrl:
          defaultZone: http://localhost:8083/eureka/
server:
  port: 8087
```
这两个服务建立并开启之后，新建一个服务端，然后通过ribbon访问`eurekamovie`这个服务名称，这样请求就会均匀的分布在这两个服务上。

建立一个RestTemplate自动注解类
```java
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
    
    @Autowired
    LoadBalancerClient loadBalancerClient;
```
最后在Controller里面的方法里面通过调用该服务名称完成调用。
```java
        RestTemplate restTemplate = new RestTemplate();
        ServiceInstance instance = loadBalancerClient.choose("EUREKAMOVIE");
        URI uri = instance.getUri();
        User user =this.restTemplate().getForEntity(uri+"aa",User.class).getBody();
```
最后可以在另两个服务中可以看到请求按照一个一次的方式均匀的分布在这两个实例中。另外就是如果需要自己编写负载均衡的规则的话。可以参照如下代码：
```java
@Configuration
public class RibbonConfig {
    @Bean
    public IRule ribbonule(){
        return new RandomRule();
    }
}
```
自定义的规则只需要返回一个IRule规则即可。