---
title: Java集合学习之HashSet
date: 2019-04-04 23:30:28
tags: [Java]
categories: Java
---
## 简介
在一般的使用中，HashSet经常用于数据的去重，例如我们有一个List，这个List里面有一些重复的数据，于是我们便可以这样操作
```java
List<String> list = new ArrayList<>();
list.add("a");
list.add("b");
list.add("c");
list.add("a");
Set<String> set =new HashSet<>();
set.addAll(list);
```
此时，在Set里面，只会有一个`a`元素。

## 底层
其实`HashSet`的底层是一个`HashMap`，`HashSet`的去重使用了`HashMap`的`Key`。
如图所示：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/HashSet%E5%BA%95%E5%B1%82.png)

```java
    /**
     * Adds the specified element to this set if it is not already present.
     * More formally, adds the specified element <tt>e</tt> to this set if
     * this set contains no element <tt>e2</tt> such that
     * <tt>(e==null&nbsp;?&nbsp;e2==null&nbsp;:&nbsp;e.equals(e2))</tt>.
     * If this set already contains the element, the call leaves the set
     * unchanged and returns <tt>false</tt>.
     *
     * @param e element to be added to this set
     * @return <tt>true</tt> if this set did not already contain the specified
     * element
     */
    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }

```
HashSet的`add`方法是向一个map里面放入元素，而`HashMap`则是不允许键重复，所以就可以确保在`HashMap`上的键都是不重复的。

## HashMap是如何确保每一个对象都只有一个的呢?
首先当调用`HashSet`的add方法的时候，其实是调用HashMap的`put`方法，
```java
 /**
     * Associates the specified value with the specified key in this map.
     * If the map previously contained a mapping for the key, the old
     * value is replaced.
     *
     * @param key key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     * @return the previous value associated with <tt>key</tt>, or
     *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
     *         (A <tt>null</tt> return can also indicate that the map
     *         previously associated <tt>null</tt> with <tt>key</tt>.)
     */
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```

这个方法无非是先将这个key进行hash，然后再调用`putVal`方法进行保存，
```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```
这个是`HashMap`的底层方法，当首次传入值的时候，
>    if ((tab = table) == null || (n = tab.length) == 0){
>            n = (tab = resize()).length;}

如果table未空就进行初始化，如果不为空则执行下面的代码
```java
Node<K,V> e; K k;
if (p.hash == hash &&
    ((k = p.key) == key || (key != null && key.equals(k))))
    e = p;
else if (p instanceof TreeNode)
    e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
else {
    for (int binCount = 0; ; ++binCount) {
        if ((e = p.next) == null) {
            p.next = newNode(hash, key, value, null);
            if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                treeifyBin(tab, hash);
            break;
        }
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
            break;
        p = e;
    }
}
```
在HashMap里面，有一个数组`table`存放着所有的`key`，而 HashMap 定位下标的方式就是通过`(n - 1) & hash`。

当HashMap发现该下标的值是`null`，就会直接将入参的`key`和`value`疯转成一个Node保存进去，如果发现不是`null`，则`HashMap`认为发生了`HashMap`碰撞，于时进行如下判断:


如果新传入的一个key在`HashMap`中已经存在，则`HashMap`会直接将旧的key的value替换掉。否则就会进行新增
HashMap在判断一个key是否相等会采取以下措施：

**如果两个Key的hash不同，则HashMap直接会判断key不等**
1. Key和hash完全相同
> 第一次传入key=a，Hash值是1，value是100;  
> 第二次传入key=a，Hash值是1，value是101;

此时hashMap会认为新传入的key已经存在，所以会将旧的value替换为新的value


**产生Hash碰撞，也就是两个key的hash都是一样的，那么就会通过key是否相同或者equals方法判断对象是否相等了**
2. Key不同，hash相同 

HashMap会判断传入的key、以及key的hash值，如果相等则认为该键以相等，例如:
> 第一次传入key:a，Hash值是1;  
> 第二次传入key:b，Hash值是1;

这个时候由于key不同，hashMap还会通过equals继续判断。

3. 由于第二次传入的key是`b`，但是他们的Key并不相等，此时`HashMap`就会调用他们的`equals`方法，如果通过`equals`方法判断的是相同对象，则也会认为是同一个key。


此时如果判断新增的key确实不存在就会在当前的table位置通过链表地址方法辛增一个key了。

那么到这里就可以看到，其实`HashSet`就是完全的利用了`HashMap`的键的特性来进行去重。

## Iterator方法
```java
   /**
     * Returns an iterator over the elements in this set.  The elements
     * are returned in no particular order.
     *
     * @return an Iterator over the elements in this set
     * @see ConcurrentModificationException
     */
    public Iterator<E> iterator() {
        return map.keySet().iterator();
    }
```
其实都是利用了`hashMap`的一些方法来实现

## 线程不安全
由于HshMap是非线程安全的，自然HashSet也不是一个线程安全的。测试代码如下:
```java
public class HashSetTest implements Runnable{
    Set<Integer> set =new HashSet<Integer>();
    public void run() {
        for(int i=0 ;i<10000;i++){
            set.add(i);
        }
    }
}
public class HashSetTestMain {
    public static void main(String[] args) throws InterruptedException {
        HashSetTest hashSetTest =new HashSetTest();
        Thread t1 = new Thread(hashSetTest);
        Thread t2 = new Thread(hashSetTest);
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(hashSetTest.set.size());
        Set<Integer> set =new HashSet<Integer>();
        for(Integer i :hashSetTest.set){
            if(set.contains(i)){
                System.out.println(i);
            }else{
                set.add(i);
            }
        }
    }
}

```
可以看到打印出来的结果会多于`10000`，这是因为在上面也说到过的，HashMap在判断一个key是否相同以及后续新增节点的时候并非是一个原子性的，所以就有可能会导致`t1`线程刚好判断10不在hashMap中，准备新增一个节点为10。结果此时t1被挂起，t2执行，但是t2也判断了10不在hashMap中，也准备新增，那么此时就会出现新增了两个一摸一样的Key。这样就会导致`Set`集合中出现了重复的数据。