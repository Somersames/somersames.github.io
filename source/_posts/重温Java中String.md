---
title: 重温Java中String
date: 2020-04-06 21:41:39
tags: [Java]
categories: [Java,JDK]
---
本文的内容都是基于 JDK1.8 来写的，主要是复习下 String 类的设计。
## 简介
String 是一个用于存储字符串的类，其内部是通过 char 数组来实现，在 Java 中，1byte = 8bit，1char = 2byte, 所以在 Java 中，String 的code point是16位。
String 类是由 final 关键字来修饰的，因此表明该类不可以被继承，同时 String 又实现了 Serializable、Comparable、CharSequence接口，表明 String 可以被序列化，以及使用cpmpareTo来比较两个字符串的值。

## 字符集编码
### 内码
内码指的是程序内部自己使用的字符集，java中是以 `UTF-16` 来表示的
> The Java programming language represents text in sequences of 16-bit code units, using the UTF-16 encoding.

UTF-16最多可以表示 65535 种字符，那么不在 65535 之内的字符，该如何表示呢，这时候就需要用两个字节来表示这个字符。以😁这个emoje为例子：其Code的编码是`1F601`。
```java
具体方法是：

Code Point减去0x10000， 得到的值是长度为20bit（不足的补0）；
将得到数值的高位的10比特的值 加上0xD800的前6位得到第一个Code Unit。
步骤1得到数值的低位的10比特的值 加上0xDC00的前6位得到第二个Code Unit。
```
于是计算方法如下：
```java
x = 1F601 - 0x10000 = 1111011000000001
不满20位的补0；
所以得到：00001111011000000001。
0xD800：1101100000000000
0xDC00：1101110000000000
于是高位的代理就是：1101100000111101
于是低位的代理就是：1101111000000001
最终得到：d83d,de01
```
在 String 的代码中体现如下：
```java
 public static int codePointAt(CharSequence seq, int index) {
    char c1 = seq.charAt(index);
    if (isHighSurrogate(c1) && ++index < seq.length()) {
        char c2 = seq.charAt(index);
        if (isLowSurrogate(c2)) {
            return toCodePoint(c1, c2);
        }
    }
    return c1;
}

public static int toCodePoint(char high, char low) {
    // Optimized form of:
    // return ((high - MIN_HIGH_SURROGATE) << 10)
    //         + (low - MIN_LOW_SURROGATE)
    //         + MIN_SUPPLEMENTARY_CODE_POINT;
    return ((high << 10) + low) + (MIN_SUPPLEMENTARY_CODE_POINT
                                    - (MIN_HIGH_SURROGATE << 10)
                                    - MIN_LOW_SURROGATE);
}
```
> 注意这个high << 10是因为code转成16进制的时候，还补了4个0
当 String 判断是否是一个 code point 的时候，首先会判断是否是高位代理，是的话在判断下一个字节是否是低位代理，是的就通过`toCodePoint`来推导出原来的code。
## readResolve方法
这个方法是为了保证序列化的时候，避免生成多个 String 对象，首先当一个对象被反序列化的时候，其调用链如下:
>ObjectInputStream -> readObject() -> readObject0(boolean unshared) -> readOrdinaryObject(boolean unshared) 

在`readOrdinaryObject`中，主要是需要注意如下两段代码：
```java
Object obj;
try {
    obj = desc.isInstantiable() ? desc.newInstance() : null;
} catch (Exception ex) {
    throw (IOException) new InvalidClassException(
        desc.forClass().getName(),
        "unable to create instance").initCause(ex);
}

if (obj != null &&
    handles.lookupException(passHandle) == null &&
    desc.hasReadResolveMethod())
{
    Object rep = desc.invokeReadResolve(obj);
    if (unshared && rep.getClass().isArray()) {
        rep = cloneArray(rep);
    }
    if (rep != obj) {
        // Filter the replacement object
        if (rep != null) {
            if (rep.getClass().isArray()) {
                filterCheck(rep.getClass(), Array.getLength(rep));
            } else {
                filterCheck(rep.getClass(), -1);
            }
        }
        handles.setObject(passHandle, obj = rep);
    }
}
ObjectStreamClass：
readResolveMethod = getInheritableMethod(cl, "readResolve", null, Object.class);
```
如果没有重写`readResolve`的话，那么此时便会直接返回`newInstance` 生成的新对象了。


## final 的使用
在 String 中，可以看到需要地方使用到了 final 字段，例如：
```java
private final char value[];
private static final ObjectStreamField[] serialPersistentFields = new ObjectStreamField[0];
```
这些都好理解，但是其中有一个方法里面的变量名却也使用了final，如下：
```java
private int indexOfSupplementary(int ch, int fromIndex) {
    if (Character.isValidCodePoint(ch)) {
        final char[] value = this.value;
        final char hi = Character.highSurrogate(ch);
        final char lo = Character.lowSurrogate(ch);
        final int max = value.length - 1;
        for (int i = fromIndex; i < max; i++) {
            if (value[i] == hi && value[i + 1] == lo) {
                return i;
            }
        }
    }
    return -1;
}
```
在这里我个人感觉其实是没什么作用的，因为方法内部的`value`会随着`this.value`的变化而变化(一般不可能)，所以这里的final，也就`hi`和`lo`以及`max`有作用，可以保证同一个方法内部，
在多线程的竞态条件下不改变。