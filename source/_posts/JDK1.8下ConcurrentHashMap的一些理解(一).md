---
title: JDK1.8下ConcurrentHashMap的一些理解(一)
date: 2019-05-13 23:26:02
tags: Java
categories: java
---
在JDK1.8里面，`ConcurrentHashMap`在put方法里面已经将分段锁移除了，转而是CAS锁和synchronized


`ConcurrentHashMap`是Java里面同时兼顾性能和线程安全的一个键值对集合，同属于键值对的集合还有`HashTable`以及`HashMap`，
`HashTable`是一个线程安全的类，因为它的所有`public`方法都被`synchronized`修饰，这样就导致了一个问题，就是效率太低。

虽然`HashMap`在`JDK1.8`的并发场景下触发扩容时不会出现成环了，但是会出现数据丢失的情况。
所以如果需要在多线程的情况下(多读少写))使用Map集合的话，`ConcurrentHashMap`是一个不错的选择。


`ConcurrentHashMap`在JDK1.8的时候将put()方法中的分段锁`Segment`移除，转而采用一种`CAS`锁和`synchronized`来实现插入方法的线程安全。
如下代码：

```java
/** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
       //省略相关代码
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) {
                    //省略相关代码
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
```

可以看到在`JDK1.8`里面，`ConcurrentHashMap`是直接采用`数组`+`链表`+`红黑树`来实现，时间复杂度在O(1)和O(n)之间，如果链表转化为红黑树了，那么就是O(1)到O(nlogn)。
在这里值得一提的是，`ConcurrentHashMap`会判断`tabAt(tab, i = (n - 1) & hash)`是不是 null，是的话就直接采用`CAS`进行插入，而如果不为空的话，则是`synchronized`锁住当前`Node`的首节点，这是因为当该`Node`不为空的时候，证明了此时出现了`Hash`碰撞，就会涉及到`链表`的尾节点新增或者`红黑树`的节点新增以及`红黑树`的平衡，这些操作自然都是非原子性的。


从而导致无法使用`CAS`，当`Node`的当前下标为null的时候，由于只是涉及数组的新增，所以用`CAS`即可。
> 因为CAS是一种基于版本控制的方式来实现，而碰撞之后的操作太多，所以直接用`synchronized`比较合适。


### ConcurrentHashMap在迭代时和HashMap的区别
当一个集合在迭代的时候如果动态的添加或者删除元素，那么就会抛出`Concurrentmodificationexception`，但是在`ConcurrentHashMap`里面却不会，例如如下代码:
```java
public static void main(String[] args) {
    Map<String,String> map = new ConcurrentHashMap<String, String>();
    map.put("a","a1");
    map.put("b","b1");
    map.put("c","c1");
    map.put("d","d1");
    map.put("e","e1");
    Iterator<String> iterator = map.keySet().iterator();
    while (iterator.hasNext()){
        String it = iterator.next();
        if("b".equals(it)){
            map.remove("d");
        }
        System.out.println(it);
    }
}

控制台打印如下：
a
b
c
e
```

而当你把`ConcurrentHashMap`换成`HashMap`的时候，控制台就会抛出一个异常:
```java
Exception in thread "main" a
b
java.util.ConcurrentModificationException
	at java.util.HashMap$HashIterator.nextNode(HashMap.java:1442)
	at java.util.HashMap$KeyIterator.next(HashMap.java:1466)
	at xyz.somersames.ListTest.main(ListTest.java:22)
```

原因在于`ConcurrentHashMap`的`next`方法并不会去检查`modCount`和`expectedModCount`，但是会检查下一个节点是不是为空
```java
  if ((p = next) == null)
    throw new NoSuchElementException();
```
当我们进行remove的时候，`ConcurrentHashMap`会直接通过修改指针的方式来进行移除操作，同样的，也会锁住`数组`的头节点直至移除结束，所以在同一个时刻，只会有一个线程对`当前数组下标的所有节点`进行操作。


但是在`HashMap`里面，`next`方法会进行一个check，而remove操作会修改`modCount`，导致`modCount`和`expectedModCount`不相等，所以就会导致
`ConcurrentModificationException`

稍微修改下代码:
```java
public static void main(String[] args) {
    Map<String,String> map = new ConcurrentHashMap<String, String>();
    map.put("a","a1");
    map.put("b","b1");
    map.put("c","c1");
    map.put("d","d1");
    map.put("e","e1");
    Iterator<String> iterator = map.keySet().iterator();
    while (iterator.hasNext()){
        if("b".equals(iterator.next())){
            map.remove("d");
        }
        System.out.println(iterator.next());
    }
}
控制台打印如下:
b
d
Exception in thread "main" java.util.NoSuchElementException
	at java.util.concurrent.ConcurrentHashMap$KeyIterator.next(ConcurrentHashMap.java:3416)
	at com.xzh.ssmtest.ListTest.main(ListTest.java:25)
```
### 并发下的处理
由于每一个`Node`的首节点都会被`synchronized`修饰，从而将一个元素的新增转化为一个原子操作，同时`Node`的`value`和`next`都是由`volatile`关键字进行修饰，从而可以保证可见性。