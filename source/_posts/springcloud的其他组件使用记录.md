---
title: springcloud的其他组件使用记录之Config(四)
date: 2018-04-22 23:40:20
tags: [web后端,SpringCloud]
categories: [Java,SpringCloud]
---
在使用Spring Config的时候遇到一些坑，在这里记录下，顺便梳理下这个使用。

## Spring Cloud Config
使用Spring Config来配合git做一个配置文件管理，需要一个Config的服务端和一个Config的客户端，服务端主要是和git仓库进行一个连接，而config的客户端是连接服务端来刷新配置服务的。
在Spring Cloud Config里面客户端需要使用`Spring4.0`出现的一个注解`@Value`配合一起使用

## Spring Config服务端
需要引入一下几个文件：
```xml
     <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>
      <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>
         <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
```
新建一个application.yml
```yml
spring:
  application:
    name: microservice-server  # name可以随便填写，代表这个服务的ServiceId
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/somersames/sprincloud-config.git  # git的地址
          username: yourname
          password: yourpassword
server:
  port: 8099 # 服务开启的端口
```
新建一个启动类：
```java
@SpringBootApplication
@EnableConfigServer
public class ConfigApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigApplication.class);
    }
}
```
`@EnableConfigServer`代表的是将这个微服务作为Config的服务器


## 配置文件

随后在git服务器中新建几个文件，并且按照peoperties的格式输入内容，例如`profile=ad`如下：
```text
microservice-foo.properties
microservice-foo-dev.properties
microservice-foo-test.properties
```
分别代表的是默认配置和开发环境以及测试环境的配置文件。


## 开启服务端

运行服务端，然后访问URL，其中URL的格式为localhost:port/默认的项目名称/分支.格式
![](json格式.PNG)
这个URL的格式有很多种格式，具体的可以百度之后再自己尝试
```json
{"name":"microservice-foo","profiles":["dev.json"],"label":"master","version":"7ac2a341dfe7959b809b7d5ec70b980970208b91","state":null,"propertySources":[{"name":"https://gitee.com/somersames/sprincloud-config.git/microservice-foo.properties","source":{"profile":"default-1.0-changeewafasf"}}]}
```


## 添加客户端
客户端不负责直接和git进行通信，而是直接和Config的服务端进行通信获取最新的数据

新建一个工程并且添加一个配置文件`bootstrap.yml`，没错，是bootstrap.yml，然后再新建一个配置文件`application.yml`.
在bootstrap.yml中添加内容：
```yml
spring:
  application:
    name: microservice-foo # 这里的名称填写项目的名称，也就是在之前获取的json里面的那个name
  cloud:
    config:
      uri: http://localhost:8099/  # 填写Config服务端地址
      profile: dev  #项目环境
      label: master  #项目分支
```

在Application.yml里面添加内容：
```yml
server:
  port: 8088 # 服务开启的端口，任意即可
```


新建一个Controller
```java
@RestController
@RequestMapping("config")
public class FeignController {

    @Value("${profile}")  // 这里的profile不是随便取得，这里取得是上述josn字符串里面的propertySources 下的 source 里面的那个键，在这个例子里面就是profile,
    private String profile;

    @RequestMapping(value = "/profile",method = RequestMethod.GET)
    public String hello(){
        return this.profile;
    }
}
```
其他无需改动，然后访问`localhost:8099/config/profile`，可以看到如下结果：
![](config.PNG)


## 刷新

有时候需要比如动态刷新git的最新配置的话，需要引入一个新的包：
```xml
  <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
            <version>1.2.7.RELEASE</version>  <!--根据自己的版本自己选择合适的 -->
        </dependency>
```
然后在Controller里面添加注解`@RefreshScope`,但是需要注意的是高版本需要在bootstrap.yml里面添加一个配置**`management:security:enabled: false`**否则会导致修改之后请求`refresh`刷新不出来。

自己通过git修改那个文件之后继续如下操作：

然后POSTMAN发出一个POST请求，
![](POST刷新.PNG)

如果在高版本里面不添加那个配置会导致刷新不出来，如果自己刷新不出来，请尝试添加那个配置,另外那个额外添加的配置需要按照格式自己调整下。刷新之后如下：

![](刷新.PNG)

那个cvcvcv就是刚刚修改的内容


这个就差不多结束了