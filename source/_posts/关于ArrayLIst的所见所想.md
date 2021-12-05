---
title: 关于ArrayList的所见所想
date: 2017-11-07 21:46:25
tags: [Java]
categories: [Java,集合]
---

# 关于ArrrayList扩容：
 > 今天在面试的时候面试官提到过ArrayList扩容是原来的1.5倍加一，但是我看jdk1.8的时候是显示为1.5倍
 ### 查看源码：
 ```java
 //Java1.8的扩容方法
 private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
 ```
 那么ArrayList会是扩充1.5倍之后加一呢？
 > 于是翻看以前的jdk源码，发现在jdk1.6的时候其代码确实是+1了
 ```java
 // java1.6版本的扩容方法
 public void ensureCapacity(int minCapacity) {
        modCount++;
        int oldCapacity = elementData.length;
        if (minCapacity > oldCapacity) {
            Object oldData[] = elementData;
            int newCapacity = (oldCapacity * 3)/2 + 1;
            if (newCapacity < minCapacity)
                newCapacity = minCapacity;
            // minCapacity is usually close to size, so this is a win:
            elementData = Arrays.copyOf(elementData, newCapacity);
        }
    }
 ```
# 关于其扩容具体做法：
### jdk1.8
 其次再jdk1.8中的ArrayList的一些分析：
 首先需要向List中添加元素的时候：
 > 调用 boolean add(E e);
 这个方法首先会调用 ensureCapacityInternal(int)方法
 ```java
    private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }
 ```
 当第一次调用的时候因为在构造方法里面已经将设置为'elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA'
 所以第一次调用的时候是直接接着调用'private void ensureExplicitCapacity(int minCapacity)'方法。
 但是这里会发现一个问题就是jdk1.8并未提供初始的默认值，
 在jdk1.6中ArrayList的构造方法：
 ```java
  public ArrayList() {
        this(10);
    }
    public ArrayList(int initialCapacity) {
        super();
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        this.elementData = new Object[initialCapacity];
    }
 ```
 但是在jdk1.8中却并未有默认的初始值：
 ```java
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
 ```
 而是提供了一个修改容量的方法：
 ```java
 public void ensureCapacity(int minCapacity) {
        int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
            // any size if not default element table
            ? 0
            // larger than default for default empty table. It's already
            // supposed to be at default size.
            : DEFAULT_CAPACITY;

        if (minCapacity > minExpand) {
            ensureExplicitCapacity(minCapacity);
        }
    }
 ```


