---
title: 记一次parallelStream错误使用导致的NullPointerException
date: 2020-05-11 00:21:13
tags: [Java]
categories: [Java,Lambda]
---

## parallelStream
在 Java8 中，新增了一个很有用的功能就是 `流`，这个功能可以使我们可以快速的写出优雅的代码，其中 `stream` 是一个串行流（说法可能有误...就是不会采取多线程来进行处理）。还有一种就是 `parallelStream` 采用 `ForkJoinPool` 来实现并发，加快执行效率。


所以在使用 `parallelStream` 的时候一定要注意线程安全的问题，首先看一段代码：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/%E8%AE%B0%E4%B8%80%E6%AC%A1parallelStream%E9%94%99%E8%AF%AF%E4%BD%BF%E7%94%A8%E5%AF%BC%E8%87%B4%E7%9A%84NullPointerException/%E4%BB%A3%E7%A0%81%E7%A4%BA%E4%BE%8B.png)


在这段代码中，是首先判断 `dataListList` 里面对象的 `name` 属性是否是偶数，是的话则添加至偶数List，反之则添加至奇数List。

然后开始测试这段代码：
```java
public static void main(String[] args) {
    for(;;){
      DataListTest.test();
  }
}
```

运行一段时间你就会发现，会出现 `NullPointerException`，这是因为在之前的 `forEach` 里面 ArrayList 是一个非线程安全的集合，而 `parallelStream` 是一个多线程的流，所以就会导致 `ArrayList` 在并发插入的时候，会出现部分元素是null的情况。具体原因如下：

## ArrayList并发不安全
ArrayList并发不安全的点在于 `add` 方法里面没有使用锁来保证线程安全，下面是 add 的代码：
```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```
假设不慎将 `ArrayList` 用于并发的环境，那么当第一个线程读取到 size 是 0，第二个线程也读取到该 size 也是0，然后都执行完了 `ensureCapacityInternal` 方法，此时线程一执行完 `size++` 后被挂起，然后线程二也执行了 `size++`，那么此时无论哪一个线程先执行数组的赋值操作，它的值一定会被另一个线程所覆盖。



> 回到上面并发流空指针那一个例子

出现异常情况的 `List` 如下：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/%E8%AE%B0%E4%B8%80%E6%AC%A1parallelStream%E9%94%99%E8%AF%AF%E4%BD%BF%E7%94%A8%E5%AF%BC%E8%87%B4%E7%9A%84NullPointerException/List%E4%B8%BAnull.png)

然后在 `Collectors.toMap()` 方法里面最终会调用到 HashMap 的 merge 方法，而 HashMap 的 merge 方法第一行就是判断 `value` 是否为空，所以就导致了 `NullPointerException`

![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/%E8%AE%B0%E4%B8%80%E6%AC%A1parallelStream%E9%94%99%E8%AF%AF%E4%BD%BF%E7%94%A8%E5%AF%BC%E8%87%B4%E7%9A%84NullPointerException/merge%E4%B8%BA%E7%A9%BA.png)

## 总结
在使用 `parallelStream` 一定需要注意并发的安全，同时注意在重构代码的时候如果是这种并发流的话，一定要注意线程安全。