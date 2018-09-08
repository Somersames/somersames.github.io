---
title: 页面布局以及JS解析json的总结
date: 2018-03-28 23:29:22
tags: [web前端]
categories: Java
---
## 关于页面的水平垂直居中：
页面的水平垂直居中布局的话目前就我这里了解的话是又两种方法，一种是盒子布局，一种是流式布局：
盒子布局




## 关于Jquery解析JSON格式的问题
### JSON.parse()方法
在使用这个方法解析Json格式的时候一直会报错，但是传入的值却又明明是JSON格式的，所以一直在排查：
```
a='{"first":"111"}';
b="{'first':'111'}";
console.log(JSON.parse(a));
console.log(JSON.parse(b))  //在这里会报错，提示Unexpected token ' in JSON at position 1
```
这就引出了在JS中使用JSON解析Json字符串的问题了，下图是解析字符串的顺序了，在下图可以看到解析是以`"`为起点，然后是`/`，若在开始的位置没有发现这两个起始符号的话那么js会跳过这次解析直接到达末尾，然后报错，这也就解释了为什么在解析b的时候会直接抛出异常`Unexpected token '`
![](JS解析Json.PNG)

### JSON.parse()方法

这个方式是将一个Js的Object解析成一个Json格式的字符串，如下:
```
var obj={
    'a':123,
    "b":"qq"
}
console.log(JSON.stringify(obj))

输出{"a":123,"b":"qq"}
```
在这里需要注意的是在这个对象中的话，单引号会被转成双引号。

### JSON格式：
JSON格式的规范如下，而使用JSON.parse()来解析单引号的内容的话是不符合规范的。
> JSON 名称/值对
JSON 数据的书写格式是：名称/值对。

名称/值对包括字段名称（在双引号中），后面写一个冒号，然后是值：