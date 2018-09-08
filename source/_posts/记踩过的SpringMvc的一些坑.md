---
title: 记踩过的SpringMvc的一些坑
date: 2018-03-22 00:12:45
tags: [spring,web后端]
categories: Java
---
时隔一年多，再次在新公司期间接触了SpringMvc，由于之前一段时间再用Python和SpringBoot做项目，所以一时间导致SpringMvc配置中出现了好多坑，遂逐一记录：

# 关于Dao层找不到的异常
在配置的过程中这个异常出现的次数是最多的，也是最烦人的，一般是由于在Controller层中找不到Service层，然后Service层的Impl在自动装配dao的时候找不到dao，所以异常就会沿着service到达contrller层，但是总结起来，在今天的配置中遇到的情况主要又以下几种：


##### web.xml中的配置出现了错误：
在Spring5中默认xml文件是在WEB-INF中的，于是也就想着少配置一点是一点的原则，所以在web.xml中只是配置了分发器。但是今天却在其中发现了一些可能会导致Dao层找不到的原因，如下所示是我之前在web.xml中配置的一个详情：
```
<servlet>
    <servlet-name>spring-dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class><!-- 需要wenmvc这个jar包-->
    <load-on-startup>1</load-on-startup>
  </servlet>

  <servlet-mapping>
    <servlet-name>spring-dispatcher</servlet-name>
    <url-pattern>/</url-pattern>
  </servlet-mapping>
```
对，就是这么简单，所以也才导致了Dao层出错了，具体解释如下：
在StackOverFlow中发现有一个人的解释十分的独特，大意是web.xml中的配置有问题：
```
It is because the servlet-context.xml is placed inside the dispatcher servlet. Since dispatcher servlet is child of parent context, the parent context does not have the dependencies of child, So if you put the servlet-context.xml inside context param, and you must have the appServlet-servlet.xml inside the init param it will work fine.
```
于是我自己就修改了web.xml这个文件，修改之后的web.xml配置如下:
```
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>
  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>
      classpath:ApplicationContext.xml
    </param-value>
  </context-param>
  <servlet>
    <servlet-name>spring-dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>
        classpath:spring-mvc.xml
      </param-value>
    </init-param>
  </servlet>
  <servlet-mapping>
    <servlet-name>spring-dispatcher</servlet-name>
    <url-pattern>/</url-pattern>
  </servlet-mapping>
```
然后马上就运行成功了。。。

##### 这个就属于自己粗心了
在mapper配置文件中我这边是返回的一个集合。然后标签打错了，就成了`resultType`于是一直出错，后来换成了正确的`resultMap`就正确了。


##### 修改web.xml之后导致的命名空间出错了


