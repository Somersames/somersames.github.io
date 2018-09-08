---
title: HTTP请求的ContentType以及其使用范围
date: 2018-04-26 23:45:12
tags: [web后端,计算机网络]
categories: 计算机网络
---

在发送请求的是时候最需要注意的是 `Content-Type` ，因为不同的 Type 对应的则是不同类型的数据，今天正好没什么事情，所以来总结下：

## application/json
这种类型在最近几年用的比较多，主要是由于现在前后端分离，数据的请求方式可以由以前的表单提交逐渐偏向于Json的这种格式。所以这种 `application/json`格式的数据也就越来越多了。

这种数据现在一般使用的较多。
例如，在使用Jquery的时候如果没有指定dataType的话，在后端可以设置 Content-Type 为 application/json 也是可行的。
用 Java 则是 
```java
response.setHeader("Content-Type","application/json");
```

## application/x-www-form-urlencoded
这种就是最常见的一种表单提交方式，也就是常用的form提交了。如果通过表单提交并且想在表单中添加文件，则需要在 `form` 标签中加入 `enctype`属性，并且指定 `enctype` 为 `multipart/form-data`。

而表单默认的是 `application/x-www-form-urlencoded` 所以说其实当没有主动写这个属性的时候，浏览器已经帮你加上去了。

## text/plain
最常用的一个字符串传输类型