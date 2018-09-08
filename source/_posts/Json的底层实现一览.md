---
title: Json的底层实现一览
date: 2018-09-06 23:44:26
tags: [Json]
categories: Json
---
在开始了解Json的原理之前，首先看一段代码，在这里以阿里的`FastJson`为例。
```java
public class JsonRun {
    public static void main(String[] args) {
        JSONObject jsonObject =new JSONObject(true);
        jsonObject.put("id","a");
        jsonObject.put("name","b");
        System.out.println(jsonObject.toJSONString());
    }
}
```
当看到上述代码的时候，可能一般的程序员都会想到的是输出为如下`Json`串
> {"id":"a","name":"b"}
但是运行这段程序，你会发现控制台打印出来的是如下代码：
> {"name":"b","id":"a"}

那么为什么会出现这种情况呢，翻开`FastJson`的源码便知道了，首先定位到 JsonObject 这个类的构造函数，如下：
```java
public JSONObject(int initialCapacity, boolean ordered){
        if (ordered) {
            map = new LinkedHashMap<String, Object>(initialCapacity);
        } else {
            map = new HashMap<String, Object>(initialCapacity);
        }
    }
```
这里的 `ordered` 为一个构造参数，表示的是是否按照顺序添加，此处先不管，然后可以发现在阿里的FastJson中，其实默认的Json实现是一个Map，那么对于LinkedHashMap来讲，它是一个map和双向链表的整合体，所以在LinkedList中，每一个桶中的顺序是可以保证的