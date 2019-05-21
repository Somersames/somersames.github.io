---
title: JDK1.7和1.8中的HashMap区别
date: 2019-04-08 22:47:15
tags: Java集合
categories: Java
---
Jdk1.7和1.8中，HashMap的一些关键点几乎重写了。

## 主要变更点：
### 1. hash扰动算法

在jdk1.7的时候，HahMap的hash扰动算法如下:
```java
static int hash(int h) {
    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}

```
而在jdk1.8的时候，其hash算法已经修改为如下了:
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
HashMap在放入一个元素的时候，首先会获取其`HashCode`，然后将 key 的 HashCode 进行扰动，避免同一个碰撞概率太大。
如下例子。

假设一个key `a` 的 hashCode 为 `1010 1010 1110 1101 1110 1111 1000 0110`，如果不进行扰动，那么直接与table的长度 -1 进行**与**运，若table的长度是16，则计算的过程如下:
```java
1010 1010 1110 1101 1110 1111 1000 0110
0000 0000 0000 0000 0000 0000 0000 1111
-----------------------------------------
0000 0000 0000 0000 0000 0000 0000 0110
```
计算结果得出： `a` 的数组下标就是 6

但是这样就会出现一个问题，即每一次比较的都是最低位，如果某一个 key 和`a`的高位不同，低位却相同。每一次都是取最低位的几个数值进行运算，那么就会产生很严重的`hash碰撞`，所以就需要进行`hash`扰动以减少`hash碰撞`的概率。

### 以 jdk1.8 的扰动算法为例
```java
1010 1010 1110 1101 1110 1111 1000 0110
0000 0000 0000 0000 1010 1010 1110 1101  ^ >>>16
-----------------------------------------
1010 1010 1110 1101 0101 0101 0110 1000
0000 0000 0000 0000 0000 0000 0000 1111  &
-----------------------------------------
0000 0000 0000 0000 0000 0000 0000 1000   8
```

为什么进行扰动后，碰撞的概率会降低。具体的原因可以阅读这边文章
[An introduction to optimising a hashing strategy](http://vanillajava.blogspot.com/2015/09/an-introduction-to-optimising-hashing.html)



### 2. HashMap的数据结构出现了变化
在 jdk1.7的时候，HashMap是由一个数组和一个链表构成的。
插入规则如下：

1. 计算新插入的 key 的 hashCode，然后通过 hashCode 计算索引，找出该key在`Entry`中的位置，然后判断该下标是否有元素，如果没有则直接进行插入。


2. 如果有的话就按照如下规则找出是否有相同的 key：
> hash相同且key相同 
> hash相同且equals方法返回相同

若相同，则直接将当前的 value 替换原来的 value。

3. 如果最后还是未发现相同的 key ，则新建一个`Entry` ，并将头节点设置为该`Entry`。
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
        if (key == null)
            return putForNullKey(value);
        int hash = hash(key.hashCode());
        int i = indexFor(hash, table.length);
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }

        modCount++;
        addEntry(hash, key, value, i);
        return null;
    }
    
    void addEntry(int hash, K key, V value, int bucketIndex) {
        Entry<K,V> e = table[bucketIndex]; //找出原来table中的元素
        table[bucketIndex] = new Entry<>(hash, key, value, e);
        if (size++ >= threshold)
            resize(2 * table.length);
    }

      //注意，此时将该节点是作为现在的table的头节点，原来的e则是新节点的next
       /**
         * Creates new entry.
         */
        Entry(int h, K k, V v, Entry<K,V> n) {
            value = v;
            next = n;
            key = k;
            hash = h;
        }

```


以下是在 jdk1.7 的时候第三种方式插入的极简版：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/HashMap1.7.png)



#### 而在 jdk1.8 的时候，则是由一个数组加一个链表、红黑树组成
之所以这样改进，是因为在极端情况下，如果所有的元素都 hash 到了一个下标，那么这样的话，HashMap在查找元素的时候就会退化到一个链表，其时间复杂度是`O(n)`。

为了应对这种情况，HashMap在1.8的时候会判断链表上的元素，如果超过了 `8` 个，就会将链表转化为红黑树。同时在 1.8 的时候，HashMap将链表的插入方式修改为尾插入。

> 提示：修改为尾插入是为了避免在并发的情况下出现链表成环（在jdk1.7之前会出现、同时HashMap并不适用并发场景下）

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
            //......省略相关代码
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                     // 在未节点进行插入
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st。如果大于8，则会将链表转为红黑树
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

上述的变动最大点在于这两行代码：
```java
p.next = newNode(hash, key, value, null);
if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
    treeifyBin(tab, hash);
}
```
第一行是进行尾插入(1.7是头插入)

第二行是大于8会进行链表到红黑树的转化


### jdk1.7采用头节点插入导致的链表成环
虽然`HashMap`是一个非线程安全的，但是如果在 jdk1.7 版本中将HashMap用于并发环境下会出现什么情况呢？
```java

void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }

        Entry[] newTable = new Entry[newCapacity];
        transfer(newTable);
        table = newTable;
        threshold = (int)(newCapacity * loadFactor);
    }
    
// jdk1.7 扩容代码
void transfer(Entry[] newTable) {
        Entry[] src = table;
        int newCapacity = newTable.length;
        for (int j = 0; j < src.length; j++) {
            Entry<K,V> e = src[j];
            if (e != null) {
                src[j] = null;
                do {
                    Entry<K,V> next = e.next;
                    int i = indexFor(e.hash, newCapacity);
                    e.next = newTable[i];
                    newTable[i] = e;
                    e = next;
                } while (e != null);
            }
        }
    }
```
##### 第一步
------------------------------------------------------------------------

**此时假设线程二已经将hashMap扩容完毕，但是线程一还在被挂起。**
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/hashMap1.0.png)



线程一执行，此时 `e`是为1，next却是2。

-------------------------------------------------

![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/HshMap%E7%BA%BF%E7%A8%8B%E4%B8%80%E7%AC%AC%E4%B8%80%E6%AC%A1transfer.png)

线程一第一次循环执行完毕，此时的`e`是2，然后`e.next是3。

----------------------------------------------------------------
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/HashMap%E7%BA%BF%E7%A8%8B%E4%B8%80%E7%AC%AC%E4%BA%8C%E6%AC%A1%E6%89%A7%E8%A1%8C.png)




线程一第二次循环执行完毕，此时的`e`是3，然后`e.next是1`，注意此时**线程二**中，已经将 `3` 的next指向了 `1` ，所以此时`e`是3，然后 `next` 是1。

----------------------------------------------------------------
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/HashMap%E7%BA%BF%E7%A8%8B%E4%B8%80%E7%AC%AC%E4%B8%89%E6%AC%A1%E5%BE%AA%E7%8E%AF.png)

此时第三次循环完毕，由于`e`还不为空，于是进行第四次循环(**主要原因是线程二已经将`3`的next指向为`1`**)。
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/HashMap%E7%BA%BF%E7%A8%8B%E4%B8%80%E5%BE%AA%E7%8E%AF%E5%AE%8C%E6%AF%95.png)

由于`1`的 next 是 null，所以循环结束。

### jdk1.8的尾节点插入
由上面的分析可以不难发现，造成链表成环的主要原因为：多线程下，头节点插入导致原来的链表的尾节点有了`next`，所以最后会多循环一遍，从而成环。

而在jdk1.8采用的为节点插入在多线程下，顶多是另一个线程把前面一个线程 resize 的过程再重复一遍，却不会再出现链表成环。


### 多线程下通用的bug

虽然 jdk1.8 修复了链表成环这一个问题，但是多线程的情况下导致的`数据丢失`问题确实一直存在的。


所以不要尝试在多线程的情况下使用`HashMap`，如果需要用到`Map`结构的话，可以用`CurrentHashMap`或者`HashTable`