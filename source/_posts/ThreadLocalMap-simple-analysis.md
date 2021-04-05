---
title: ThreadLocalMap 简单分析
date: 2020-07-02 00:05:58
tags: [java]
categories: Java
---

## ThreadLocal 
在使用多线程的时候，如果需要保存一份线程自己私有的一部分变量，避免其他线程污染这个变量的话，一般都会自己手动 new 一个 ThreadLocal，如下代码：
```java
ThreadLocal<String> threadLocal = new ThreadLocal<>();
threadLocal.set("test");
```
当需要在某一刻使用这个变量的时候，只需要手动调用下
```java
threadLocal.get();
```  

## ThreadLocalMap
ThreadLocalMap 是 ThreadLocal 的一个静态内部类，每一个线程在创建的时候都会有这个 ThreadLocalMap 变量，它的作用就是存储每一个通过 `threadLocal.set()` 方法存入的值，本质上也是一个HashMap。

先看下它的一个内部类：
### Entry
不同于 WeakHashMap 是继承自 HashMap 的Entry，ThreadLocalMap 的Entry 则是自己的一个内部类。
```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;
    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```
在这里可以看到的是，ThreadLocalMap.Entry 是以 `ThreadLocal<?>` 为 key 的，也就是 ThreadLocalMap 它的整体设计如下：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/ThreadLocalMap/ThreadLocalMap.png)

#### 变量
`table`：表示的是一个 `Entry` 数组
`threshold`：下一次扩容的临界值，算法为 数组的长度的 2/3
`INITIAL_CAPACITY`：数组的大小，默认是 16

#### Entry扩容
`table` 的扩容是直接新建了一个 `Entry[]` 数组，将其长度设置为旧数组长度的 2 倍，然后通过一个 `for` 循环，将元素都重新移至新的 `table` 上，但是在移植的过程中，如果发现了 `Entry` 中的 Key 为空的话，那么就会直接将其 value 设置为 null 来帮助GC，避免内存的泄漏。

> 如果在移植的过程中发生了 Hash 碰撞，那么会直接将当前下标 + 1，然后判断该位置是否有元素，如果有的话，继续 + 1，直至没有元素。


### 新增过程
当调用 `threadLocal.set()` 方法的时候，会首先判断 `ThreadLocalMap` 存不存在。从而进入不同的处理流程上来。
#### ThreadLocalMap 不存在

如果不存在的话，就会调用 `createMap` 方法来进行初始化。
而 `createMap` 直接就是通过默认值来初始化一个 `ThreadLocalMap`。
```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```

#### ThreadLocalMap 存在
如果存在的话，这时直接以当前的 `ThreadLocal` 为 `Key`，通过 `Hash算法` 算出该对象在 `table` 中的下标，然后判断该下标是否有值(Entry是否为null)，如果有值的话，则判断 `Entry` 中的 `key` 是否与当前的 `ThreadLocal` 相等（地址比较），如果相等的话，则直接将 `value` 赋值，然后 return。

如果 `Entry` 中的 `key` 为 `null` 的话，则会执行 `replaceStaleEntry` 方法来找 `key`，同时在这个方法内部，还会通过 `渐进式或者启发式` 的方式来进行清除旧key操作。
先大致列出其 set 的过程：


**1.** 获取该 key 的 `threadLocalHashCode`，然后与当前 `table(Entry[]的长度) -1 ` 进行 `&` 运算，计算出下标。

##### Hash算法
`ThreadLocalMap` 的 `Hash算法` 采用的是 `斐波那契散列`，其过程是用 `0x61c88647` 累加，然后用累加的结果与当前 `table(Entry[]的长度) - 1` 进行 `&` 运算。
> `0x61c88647` 转化为十进制是 `1640531527`，
```java
private static AtomicInteger nextHashCode = new AtomicInteger();
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```
在这里的 `nextHashCode` 是一个 `AtomicInteger`，因此可以保证其原子性，而且又是私有的静态变量，所以可以尽量保证每一个 `ThreadLocal` 都会得到一个唯一的 `HashCode`。

**2.** 找出 `table[i]` 中不为 `null` 的 `Entry`，在这个过程中会一直 `i+1`，如果 `i+1` 大于数组的最大下标的话，则直接从 0 开始寻找。


**3.** 当在遍历的过程中，如果发现 `Entry 的 key` 与当前的 `ThreadLocal` 对象相等的话，则直接将值替换，如果发现某一个 `Entry` 的 `key` 为 null 的话，则直接进行 `replaceStaleEntry`。然后return。

##### replaceStaleEntry方法
该方法出现在 `set方法` 的第三步，即当判断到 `Entry` 中的 key 为 `null`，那么此时就会调用 `replaceStaleEntry` 来清除那些被回收了的 `Key`。在这个方法里面，每一个遍历都会将 `staleSlot` 赋值到 `i`。
```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    int slotToExpunge = staleSlot;

//首先将 `staleSlot` 赋值为 `i` ，通过 `i` 向前遍历，直至遍历到第一个 `Entry` 为 null，如果在遍历的过程中发现某些 `Entry` 的 `key` 为 null 的话，则将 i 赋值到 `slotToExpunge`。如果说遍历了完整的一圈没发现 key 为 null 的 Entry 的话，那么一定会在 `staleSlot` 这个地方停下来，因为进入这个方法的前提就是 `key == null`。
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;

//随后再沿着传入的 `index` 的向后进行遍历，如果此时在遍历的过程中出现了 `Hash冲突`，则直接 `value` 赋值到当前 `Entry` 的 `value` 中。同时判断在第一个遍历中的 `slotToExpunge` 是不是 `staleSlot`，如果是的话，则直接当前的 `index` 设置为`slotToExpunge`，然后调用
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {

// 因为在 set 方法的遍历中，肯定不会出现 k == key，所以此时可以判断下，如果有的话，则直接赋值，然后就可以return 了。
        if (k == key) {
            e.value = value;

            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;
//第一个循环中，没有出现 key 为null 的 Entry，则直接将 slotToExpunge 赋值为 i，
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
//expungeStaleEntry 方法会进行 rehash，直至往后遍历到第一个为 null 的 Entry停止，在此期间也会进行rehash，同时在此期间，如果遇到了 key 为 null 的 Entry，会将他们的 Entry 以及 value 都置为 null，便于 GC。
//cleanSomeSlots 方法会以 log2(len) 的循环次数进行清除旧的 Entry，如果发现 Key 为 null 的 Entry，就会再次调用expungeStaleEntry 方法继续清理旧Key
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }
        
//如果在遍历的途中，没发现 k == key，那么此时如果在向前的遍历中，也没有发现 Key 为 null 的，此时就会将slotToExpunge 设置为 i。
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }
// 赋值
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);
//如果说之前发现 key 为null 的话，那么此时slotToExpunge 肯定就不会等于 staleSlot，于是触发清除旧Key的方法。
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

**4.** 在第三步的时候，直接找到了为 `null` 的`Entry`，则直接新建一个 `Entry` 然后赋值，
**5.** 当上述步骤都处理完了以后，会通过 `cleanSomeSlots` 进行判断，如果有 `Entry` 的弱引用是否被回收了，则会进行 `rehash`，除非数组的长度已经达到了 `threshold` 并且未发现 `Entry` 的弱引用被回收了，才会进行 rehash。
```java
tab[i] = new Entry(key, value);
int sz = ++size;
// 在这里需要注意的是只有没发生弱引用的清除才会进行Rehash，因为一旦出现了清除，则会在 expungeStaleEntry 做 rehash，所以此时就不必在做一次了。
if (!cleanSomeSlots(i, sz) && sz >= threshold)
    rehash();
```


### set方法流程图
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/ThreadLocalMap/set.png)


#### expungeStaleEntry(int staleSlot)方法
该方法是 `ThreadLocal` 主要用于清除旧 Key，其代码如下：
```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // expunge entry at staleSlot
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
 //当前的Entry的key是null，则直接将value和Entry一起全部设置为 null，帮助GC
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            // 重新Rehash，不过rehash的范围是 staleSlot 到下一个为 null 的 Entry
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```
> 在这里之所以是要进行重新 `hash`，是因为一旦出现某一个 `Key` 被回收以后，会导致后面的 `Key` 无法正确的 `hash` 到数组的正确下标下，从而导致每一次的 `get` 操作都会进行一次遍历，时间复杂度由 `O(1)` 直接退化为 `O(n)`

![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/ThreadLocalMap/rehash.png)


### Get方法
`ThreadLocal` 的 `get` 方法如下：
```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

在这里需要注意的一个方法是 `getEntry` ,如果在调用的时候，发生了 `Hash冲突`，那么该方法会调用`getEntryAfterMiss`。
```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```
在这里可以发现，其实最终还是调用的 `expungeStaleEntry`，而该方法在之前已经说过了，所以 `ThreadLocal` 在 `get` 的时候，其实还会旧的 `key` 的清除。直至遍历到第一个 `e == null` 的对象，此时则表示该 `key` 不存在，于是直接返回 `null`。


## 总结

总的来说，`ThreadLocal` 在 `set` 的过程中，首先会进行 `Hash` 找出下标，如果该下标的 `Entry` 为 `null` 的话，则直接赋值，如果不为 `null` 的话，则会进行遍历，直至找出第一个不为 `null` 的 `Entry` 然后赋值。


如果在遍历的过程中，发现了某些  `Entry` 的 `Key` 为 `null` 的话，则代表可以通过清理旧的 `Entry` 来进行赋值操作，那么其过程是，首先获取到在遍历过程中 `Entry` 的 `Key` 为 `null` 的下标，记为 `staleSlot`，然后向前遍历，直至第一个 `Entry` 为 `null` 止，然后记录在遍历的过程中，最后一个 `Entry` 的 `Key` 为 `null` 的下标。
随后进行第二次的遍历，如果在往后的遍历过程中，出现了 `Entry` 的 `key` 与当前的 `key` 相等的话，则直接赋值。



然后判断如果在之前的第一个遍历中，所有的 `Entry` 的 `key` 都不为 `null` ，那么此时直接将当前下标赋值为 `旧Key` 清除的起点，随后先进行一个`渐进式清理expungeStaleEntry`，等这一步清理完毕以后，再进行一次`启发式清理cleanSomeSlots`，`cleanSomeSlots`会进行一次 `log2(n)` 次清理，以`渐进式清理expungeStaleEntry` 后的 `index` 为起点，在之后的 `log2(n)` 下标内，如果还是出现了 `key` 为 `null` 的 `Entry`，则还是会再进行 `渐进式清理expungeStaleEntry`。