---
title: 记一次使用Spring5构件Web的过程
date: 2018-03-20 18:06:50
tags: [Java]
categories: [Java,Spring]
---
由于当时在学习Spring的时候还是在一年前，那时候Spring才是刚到4.3还是4.5.然后做了一个项目之后便了解到了SpringBoot，于是一直在用SpringBoot，所以导致现在配置起来就有点忘记了。所以现在记录下此次配置的过程中所遇到的坑。

# 踩过的坑：
## 遇到在web.xml中分发请求的类找不到
第一个坑就是`org.springframework.web.servlet.DispatcherServlet`这个类一直找不到；于是在POM中添加了个各种依赖终于发现缺少`Spring Web MVC`这个依赖包。。。
```
<servlet>
    <servlet-name>spring-dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class><!-- 需要webmvc这个jar包-->
    <load-on-startup>1</load-on-startup>
  </servlet>
```

## 开启Tomcat的时候一直提示什么Cache错误
错误提示：`CacheManager No Bean Found - Not Trying to setup any Cache` 当时一直在到处查找是否开启数据库什么的，后来在StackOverFlow中查询到了问题的解决办法：
[StackOverFlow中关于Cache的错误](https://stackoverflow.com/questions/24816502/cachemanager-no-bean-found-not-trying-to-setup-any-cache)
根据他人的答案做一个总结就是：在配置文件中发现若文件中含有`<tx`的话，IDEA会自动的将引入类似`***/cache`等的`xsd`文件，所以导致了这个异常的出现。所以一般来说又两种解决办法。

#### 用IDEA的模板直接创建spring的配置文件：
过程：`File -> new -> XMLConfigurationFile -> Spring Config` 然后Spring自动的将所需要的命名空间添加到了新建的xml文件中。此时只需要添加相应的标签即可。

#### 删除导致异常的xsd

步骤：在xml配置文件中删除掉带有`cache`的的xsd

## 静态资源的配置和引用

在去年配置静态资源的时候与今年的相差无几，但是在引用的时候却出现了一些变化。
去年做题的网站的代码：
```
去年获取静态资源的配置：
    <mvc:resources mapping="/static/**" location="/static/" />
    <mvc:resources mapping="/static/**" location="/static/" />
    <mvc:resources mapping="/uploadimg/**" location="/uploadimg/" />
去年的jsp文件中的引用：

<%@ page language="java" import="java.util.*" pageEncoding="utf-8"%>
<html>
<head>
<title>解题目录</title>
<script type="text/javascript"
	src="${pageContext.request.contextPath}/static/js/jquery-3.1.1.min.js"></script>
<script type="text/javascript"
	src="${pageContext.request.contextPath }/static/js/bootstrap.js"></script>
<link rel="stylesheet"
	href="${pageContext.request.contextPath }/static/css/bootstrap.css">
<link rel="stylesheet"
	href="${pageContext.request.contextPath }/static/css/bootstrap-theme.css">
<link rel="stylesheet"
	href="${pageContext.request.contextPath }/static/css/examination.css">

```
很明显可以看到：在jsp中是首先获取到项目目录然后以绝对路径来获取静态资源


今天新建的项目：
```
xml配置文件：
    <mvc:resources mapping="/static/**" location="/WEB-INF/static/" />
    <mvc:resources mapping="/static/**" location="/WEB-INF/static/" />
    <mvc:resources mapping="/uploadimg/**" location="/WEB-INF/uploadimg/" />

jsp文件中的引用：

<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
    <link href="../static/css/TestPage.css" rel="stylesheet">
</head>

```
在这里会发现今天在配置文件中的引用多了一个`WEB-INF`然后再是static文件夹

## 关于为什么xml现在默认在WEB-INF中：

在去年的时候记得spring的配置文件都是在resources中的，但是今天却发现在resources中配置的话启动tomcat会导致一条提示就是`IOException parsing XML document from ServletContext resource [/WEB-INF/XXXXXX]`
后来查询到发现是可以修改的，默认的话直接在WEB-INF中按照前面创建xml的方式创建一个即可，但是若需要修改的话可以参照一下配置：
```
这个xml配置表示的是基本配置的Application.xml的位置
<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>
    classpath:ApplicationContext.xml
    </param-value>
</context-param>

这个是配置分发器的配置文件的位置：
<servlet>
		<servlet-name>springDefault</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>
            classpath:spring-mvc.xml
            </param-value>
		</init-param>
	</servlet>
```
可以发现在去年的时候都是制定了配置文件的路径的，所以才可以在resources中配置xml，但是再看今年的配置：
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
会发现都没配置好，所以会出现需要配置在默认的位置上。
[StackOverFlow中默认WEB-INF配置的解释](https://stackoverflow.com/questions/11652931/applicationcontext-xml-is-being-copied-to-web-inf-classes-from-src-main-resourc)

