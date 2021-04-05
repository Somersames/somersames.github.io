---
title: WeakHashMap的实现原理
date: 2020-04-22 00:29:25
tags: [Java]
categories: Java
---
# 简介
## 什么是WeakHashMap
WeakHashMap实现了 Map 接口，属于 Java 集合中的一员，其用法几乎和 HashMap 一致，但是由于它的 Entry 还继承了 WeakHashMap ，因此导致它的这个 Entry 在触发 FullGc 的时候是有可能可以被回收的。

> **以下测试，JVM参数统一为：-Xmx64M -Xms64M -XX:+PrintGCDetails**


首先上一段代码：
```java
HashMap<byte[][],byte[][]> map = new HashMap<>();
for(;;){
    byte[][] b = new byte[1024][1024];
    map.put(b,new byte[1024][1024]);
    System.gc();
}
```
很快就会看到程序抛出了 `OOM异常`

修改程序为 `WeakHashMap`
```java
WeakHashMap<byte[][],byte[][]> map = new WeakHashMap<>();
for(;;){
    byte[][] b = new byte[1024][1024];
    map.put(b,new byte[1024][1024]);
    System.gc();
}
```
你会看到程序运行了很久都没有停下，那么证明了 `WeakHashMap` 确实会对一些没有其他引用的 key 进行删除了，那么此时如果将程序的代码再次修改下：
```java
List<WeakHashMap<byte[][],byte[][]>> list = new ArrayList<>();
for(;;){
    WeakHashMap<byte[][],byte[][]> map = new WeakHashMap<>();
    byte[][] b = new byte[1024][1024];
    map.put(b,new byte[1024][1024]);
    list.add(map);
    System.gc();
}
```
此时也会很快的出现了 `OOM`异常。此时你心里可能会想的是 List 含有了对 WeakReference 的强引用，导致 GC 无法回收对象，但是这里需要注意的是，虽然 List 含有对 WeakHashMap 的强引用，但是 WeakHashMap 的Entry 是继承了 WeakReference 的。因此当 调用 `System.gc()` 的时候，WeakHashMap 的 Entry 由于没有地方引用，应该是要被回收的。

为了证实 `WeakHashMap` 的 key 确实是被 GC 回收了，下面再来看一个例子：
```java
List<WeakHashMap<byte[][],Object>> list = new ArrayList<>();
for(;;){
    WeakHashMap<byte[][],Object> map = new WeakHashMap<>();
    byte[][] b = new byte[1024][1024];
    map.put(b,new Object());
    list.add(map);
    System.gc();
}
```
此时将 WeakHashMap 的value 修改为一个小对象，你会发现再次运行的话，运行很长时间都不会出现 OOM 异常了。

那么在看了上面这么多的例子之后，说明 `WeakHashMap` 的 Entry 肯定是可以被回收的，但是具体如何被回收还是得看下源代码。

## 源代码
### eg1
首先 HashMap 因为持有其对象的强引用，所以导致 GC 无法将其回收，从而导致了 OOM 异常。

### eg2
WeakHashMap 在运行了很长时间以后，一直未出现 OOM 异常，那么说明了其内部肯定有一些的操作来回收不可达对象。那么下面就来分析为什么会被回收。
#### Entry
`WeakHashMap` 和 `HashMap` 的大致结构是一样的，但是有一个区别是，`WeakHashMap`在实现了`Map.Entry<K,V>` 的时候还继承了 `WeakReference<Object>` 。如下图所示：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/WeakHashMap/WeakHashMap-Entry.png)

可以看到，此时 `Entry<K,V>` 的 key 是继承了 `WeakReference` 的，所以在 System.gc 的时候，key是一定可以被GC的，那么还有一个就是 value 是如何被 GC 的，此时查看 put 方法：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/WeakHashMap/WeakHashMap-put.png)

在这里需要注意的是 
> Entry<K,V>[] tab = getTable();

这一行代码是会通过 `ReferenceQueue` 来判断哪些 key 是可以被回收的。
```java
private void expungeStaleEntries() {
    for (Object x; (x = queue.poll()) != null; ) {
        synchronized (queue) {
            @SuppressWarnings("unchecked")
                Entry<K,V> e = (Entry<K,V>) x;
            int i = indexFor(e.hash, table.length);

            Entry<K,V> prev = table[i];
            Entry<K,V> p = prev;
            while (p != null) {
                Entry<K,V> next = p.next;
                if (p == e) {
                    if (prev == e)
                        table[i] = next;
                    else
                        prev.next = next;
                    // Must not null out e.next;
                    // stale entries may be in use by a HashIterator
                    e.value = null; // Help GC
                    size--;
                    break;
                }
                prev = p;
                p = next;
            }
        }
    }
}
```
通过 `queue.poll()` 方法来获取被 GC 回收后的key，然后通过对key进行 hash ，取出下标，移动指针，将value 置为 null ，然后结束。至此，为什么 eg2 不会出现 OOM 的结果就出现了，那是因为 `WeakHashMap` 在进行 put 的之后，还手动调用了一次 System.gc，然后在 put 的时候调用了 `expungeStaleEntries` 方法，所以 GC 就可以把那些对象回收了。

## eg3和eg4
对于这两个例子，主要是因为在 for 循环里面是每一次直接 new 了一个 `WeakHashMap`，而且都是首次调用 put 方法，之后就没有进行任何处理。因此首次 put 进去的数据，无法触发 `expungeStaleEntries`，因此导致只有 key 可以被回收，而 value 则是无法被回收的（expungeStaleEntries 方法里面手动将 value 置为了 null，导致可以被 GC）。

