---
title: 关于Java的HashMap
date: 2017-11-09 22:40:38
tags: 
- java
- 集合
categories: Java
---
# Map接口：
1. 在map接口中的存在一个主接口：`public interface Map` 和一个内部接口：`interface Entry`
2. 其中*Map接口*主要功能是提供map的一些基本的操作，例如put,inEmpty,get等，而*Entry接口*则主要是负责遍历操作时的一些方法，例如`getKey(),getValue()，setValue`等

#HashMap实现类：
1. 在HaspMap这个类里面其实包含了很多的内部类：如下图：
```java
 Node
 HashMap
 KeySet
 KeySpliterator
 EntrySet
 Values
 HashIterator
 KeyIterator
 ValueIterator
 EntryIterator
 HashMapSpliterator
 ValueSpliterator
 EntrySpliterator
 TreeNode
```
## TreeNode类： TreeNode类是一个红黑树：里面包含的是一些红黑树的操作；
## EntrySet类： EntrySet类是提供一个HashMap的遍历方式的一个类，EntrySet类里面有iterator方法，其作用在于map的一种遍历方式：
```java
        Iterator iterator =map.entrySet().iterator();
        while (iterator.hasNext()){
            Map.Entry<Object,Object> map1 = (Map.Entry<Object, Object>) iterator.next();
            System.out.println(map1.getKey());
        }
```
这里涉及的知识是EntrySet是继承了AbstractSet间接实现了Set接口，而map.entrySet()方法返回的是一个Set<Map.Entry<K,V>>这个实例。
## 其他的暂时还不知道有什么用

# **关于HashMap的其他细节**
### 每一扩容都是2的n次方：
首先为什么每一次都是2的n次方这是由于hash函数导致的，假设一个值是1001，那么另一个值可以是1101,1011,1111，这三种情况做且运算的话结果都是1001，发生了三次碰撞，那么假设固定值是2的n次方的话，减一之后是1111，折三个数与其做且运算的话都不会相同，所以会较小碰撞次数。

### 允许null作为键/值
那么直接看代码：
```java
   public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```
很明显当我们传入的参数是null得时候hash的值是0的时候`first = tab[(n - 1) & hash]`这个first永远是tab[0];但是又由于tab的长度是2的n次方,所以tab[0]肯定是null，所以当键是null的时候得到的也是null。