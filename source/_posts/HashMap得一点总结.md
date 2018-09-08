---
title: HashMap得一点总结
date: 2018-06-06 17:21:00
tags: [Java,数据结构]
categories: Java
---
## HashMapp为什么在Hash的时候减1
在Java的Hashmap中有如下代码：
```java
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
上面有一行是 `first = tab[(n - 1) & hash]) != null` 


## HashMap为什么在传入另一个Map时加一
```java
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        int s = m.size();
        if (s > 0) {
            if (table == null) { // pre-size
                float ft = ((float)s / loadFactor) + 1.0F;
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                if (t > threshold)
                    threshold = tableSizeFor(t);
            }
            else if (s > threshold)
                resize();
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
            }
        }
    }
```
在这里加一的目的时向上取整，假设 s=10 ，那么乘以0.75之后便是7.5，加一之后再取整，便是最恰当的存储个数了。下面的判断则是说当前的map已有的key的数量是否达到了扩容的必要。如果需要扩容的话，则是直接出发扩容函数。

## HashMap通过一个键来获取值
```java
 public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
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
其主要是通过 `get(object key)` 然后再Hash这个key，最后通过hash的key来获取其值。当获取到 hash 的 key 之后再通过`tab[(n - 1) & hash`来获取保存的位置，当发现该位置为 null 的时候便直接返回 null，
若不是 null 则判断第一个位置的键的 hash 值是不是要获取的 key 的 hash 相同，是的话便返回第一个节点，不是的话就欧安段链表的下一个是不是 null，如果不是的话就判断第一个节点的下一个节点是不是红黑树，是的话直接调用红黑树的查询方法，然后返回即可。
这里的做法是让 table 第一个节点的 next 指向红黑树的头节点或者指向链表的下一个节点。

## HashMap放入键和值
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
首先还是一样，判断 tab 的长度是否是0，是的话就初始化 table，然后通过hash出来的值与 table 的长度进行与运算，找出最合适存放该key的位置，如果为 null ,则存入。
然后判断hash是否相同，另外key是否相同，是的话跳出if，然后判断当前节点是否为null，是的话就执行 `afterNodeAccess` 将该节点移动到末尾。
如果发现 p 是一个树节点的话，那么直接调用树的存入方法即可，不是的话就调用单链表的方法进行插入即可。最后还会判断是否达到了链表转树的阈值，达到了就可以转了。