---
title: Spring ContentNegotiation（内容协商）之使用篇（一）
date: 2021-04-07 00:11:11
tags: [Springboot]
categories: [Java,SpringBoot]
---


<!-- 
提纲：
一旦匹配不到就会走默认逻辑，具体的处理逻辑是
之所以配置文件请求url为json1，也可以返回json，是因为没有指定默认的返回格式，但是spring的默认是 */*，而spring的 json 的 convert 恰好又可以解析这种格式，所以就会导致除非可以匹配到，没有匹配到的全部都会转为json

当配置了第二种方式以后会通过 ParameterContentNegotiationStrategy 来解析Parameter

发现的另一个坑
不要开启 @EnableMVC，否则会导致mvc的部分配置失效

 -->
# 背景
随着业务系统的成熟，如果你的项目正好是公司的中台战略之一，但是下游系统的接收方式不统一，这一种情况在一些老的公司系统架构总经常出现，如果下游系统不方便兼容，那么就需要中台系统对外提供各种不同格式返回报文

## 内容协商
简单说就是服务提供方根据客户端所支持的格式来返回对应的报文，在 Spring 中，REST API 基本上都是以 json 格式进行返回，而如果需要一个接口即支持 json，又支持其他格式，开发和维护多套代码显然是不合理的，而 Spring 又恰好提供了该功能，那便是ContentNegotiation


在 Spring 中，决定一个数据是以 json、xml 还是 html 的方式返回有三种方式，分别如下：
> 1：favorPathExtension 后缀模式，例如：xxx.json，xxx.xml
2：favorParameter format模式，例如：xxx?format=json,xxx?format=xml,
3：通过请求的 Accept 来决定返回的值

在这三种模式中，前面两种模式都是关闭，如果需要打开，可以通过以下方式来开启
1：重写 `WebMvcConfigurer`(Spring5.X以后推荐的实现类) 的 `configureContentNegotiation` 来设置为 true 即可
2：设置 spring.mvc.contentnegotiation.favor-path-extension=true 或者 pring.mvc.contentnegotiation.favor-parameter=true

> tips:如果是使用 Spring2.X以上的版本，不要开启 @EnableWebMvc 注解，否则会导致你的配置无效，如果需要开启该注解，则只能使用方法一重写 WebMvcConfigurer 了
并且还需要说明一点的是通过配置文件开启的话，需要设置 useSuffixPattern 为 true，重写 `configureContentNegotiation` 的已经默认为 true 了

# 三种模式
## 1：favorPathExtension 后缀模式
```yml
server:
  port: 8081
spring:
  mvc:
    contentnegotiation:
      favor-path-extension: true
      media-types:
        json: application/json
```
favor-path-extension 表示是否开启后缀匹配，media-types 表示后缀以何种方式进行解析，在这里需要注意一下一定是需要有对应的 HttpMessageConvert 才能解析，否则是会提示 `406 Could not find acceptable representation`

> 在 Spring 中已经默认含有json解析的 HttpMessageConvert，所以是可以直接解析的，如果需要支持解析 xml，可以引入 xml 包


```xml
<dependency>
	<groupId>com.fasterxml.jackson.dataformat</groupId>
	<artifactId>jackson-dataformat-xml</artifactId>
</dependency>
```

当开启了后缀模式以后，返回的文本类型会根据你的入参做不同的处理，.json 会返回 json 格式的数据，.xml 会返回 xml 格式的数据，当然也可以自定义一个 HttpMessageConverter 来自定义的返回文本格式

```http
GET localhost:8081/controller/advice/decrypt.json

{
    "name": "a",
    "age": 1,
    "date": null
}

GET localhost:8081/controller/advice/decrypt.xml

<Advice>
    <name>a</name>
    <age>1</age>
    <date/>
</Advice>
```

## 2：favorParameter
这种模式下是通过在 url 中通过一个参数来区分如何解析的，spring中已经默认这个关键字是 `format`

修改配置文件如下：
```yml
server:
  port: 8081
spring:
  mvc:
    contentnegotiation:
      favor-parameter: true

```
```http
GET localhost:8081/controller/advice/decrypt?format=json

{
    "name": "a",
    "age": 1,
    "date": null
}

GET localhost:8081/controller/advice/decrypt?format=xml

<Advice>
    <name>a</name>
    <age>1</age>
    <date/>
</Advice>
```
当然也可以自己修改 parameter 的关键字，只需要在配置文件中调整下即可
```yml
parameter-name: meida_type
```
此时再次请求的时候 parameter 就需要调整为 meida_type，否则就会以默认的方式去解析返回的文本信息



## Accept解析
这种就是默认的一种解析方式，无需进行任何配置，Spring 就是默认以这种模式进行解析的
GET请求
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/talkaboutHttpMeesageConvert/get_json.png)

XML请求
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/talkaboutHttpMeesageConvert/get_xml.png)

# 总结
本文只是简单的介绍了如何使用，后续会介绍原理篇