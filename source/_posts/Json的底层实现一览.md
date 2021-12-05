---
title: Json的底层实现一览
date: 2018-09-06 23:44:26
tags: [Json]
categories: [Java,Json]
---
在开始了解Json的原理之前，首先看一段代码，在这里以阿里的`FastJson`为例。
```java
public class JsonRun {
    public static void main(String[] args) {
        JSONObject jsonObject =new JSONObject();
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
这里的 `ordered` 为一个构造参数，表示的是是否按照顺序添加，此处先不管，然后可以发现在阿里的FastJson中，其实默认的Json实现是一个Map，那么对于LinkedHashMap来讲，它是一个map和双向链表的整合体，所以在LinkedList中，每一个Node都会有一个前指针和一个后指针

# HashMap

LinkedHashMap 是一个HashMap的变种，大家都知道，一个HashMap是由一个桶和一个桶后面的节点组成的，而桶其实是一个数组，每一个桶的索引所对应的值都是由`Hash()`函数计算得出的。那么这样就会导致桶的元素是一个乱序的存储的，例如在本段代码中的`id`和`name`，它们所在的桶索引可能是:
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20180909203451.png)

这样就导致了一个问题，就是Json的键的顺序是无法保证的，那么既然HashMap是无法保证的，为什么LinkedHashMap却可以保证顺序。
## LinkedHashMap

翻开LinkedHashMap的源码可以发现，在其节点类里面，LinkedHashMap在 HashMap的Entry基础上又添加了一个`before`和`after`指针，
```java
  static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
```

那么这两个指针就是双向链表的指针。有了这两个指针之后，每一个新插入的节点都会知道他的前驱结点和后置节点，那么对于LinkedHashMap的插入顺序就会有保证了。所以其对应的数据结构如图：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E9%93%BE%E8%A1%A8%E4%B9%8B%E5%90%8E.png)

在这个结构里面，桶索引是`id`的第一个节点是一个头节点，在新插入`name`的时候，LinkedHashMap会将head节点的`after`指针指向name，所以虽然这是一个HashMap，但是它的顺序还是可以保证的。

## LinkedHashMap的迭代

区别于HashMap以索引的方式进行迭代，`LinkedHashMap`是以链表的指针进行迭代的，如以下代码所示：
```java
abstract class LinkedHashIterator {
        LinkedHashMap.Entry<K,V> next;
        LinkedHashMap.Entry<K,V> current;
        int expectedModCount;

        LinkedHashIterator() {
            next = head;
            expectedModCount = modCount;
            current = null;
        }


final LinkedHashMap.Entry<K,V> nextNode() {
            LinkedHashMap.Entry<K,V> e = next;  //next就是head节点
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (e == null)
                throw new NoSuchElementException();
            current = e;
            next = e.after; //此处每一次的迭代都是链表的after
            return e;
        }
```
可以看到在每一次迭代的时候LinkedHashMap都是以链表的next节点作为下一个迭代，那么HashMap呢？

## HashMap的迭代
```java
abstract class HashIterator {
        Node<K,V> next;        // next entry to return
        Node<K,V> current;     // current entry
        int expectedModCount;  // for fast-fail
        int index;             // current slot

HashIterator() {
            expectedModCount = modCount;
            Node<K,V>[] t = table;
            current = next = null;
            index = 0;
            if (t != null && size > 0) { // advance to first entry
                do {} while (index < t.length && (next = t[index++]) == null);
            }
        }


final Node<K,V> nextNode() {
            Node<K,V>[] t;
            Node<K,V> e = next;
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (e == null)
                throw new NoSuchElementException();
            if ((next = (current = e).next) == null && (t = table) != null) {
                do {} while (index < t.length && (next = t[index++]) == null);
            }
            return e;
        }
```

注意这一段代码
>  if (t != null && size > 0) { // advance to first entry
                do {} while (index < t.length && (next = t[index++]) == null);
    }


这一段代码的作用是找出`table[]`中第一个不为null的桶，所以其实HashMap的迭代就是依据桶中的顺序来的，但是LinkedHashMap则是按找链表的顺序来的。


# 总结
其实每一个java的设计都是很精妙的...