---
title: 关于Leetcode上判断位数的解法
date: 2018-08-17 23:39:07
tags: [Leetcode,算法]
categories: 算法
---
在Leetcode上有一道算法题目判断最后一位是不是一位的，题目的意思是当在一个数组中如果存在`10`或者`00`，那么这个就是一个连续的。这个数组的最后一位永远都是`0`。

> We have two special characters. The first character can be represented by one bit 0. The second character can be represented by two bits (10 or 11).
Now given a string represented by several bits. Return whether the last character must be a one-bit character or not. The given string will always end with a zero.


# 思路：
刚开始错误的理解题目的意思了，导致一直在纠结数组的最后两位和三位，后来看了答案之后觉得自己的思路没有想到电子上，所以在此记录一下。

## 解法一：
这一题有一个明显的规律，也就是当 bits[i] 为 1 的时候，那么他就可以不管后面的一位，可以直接后移两位来判断。当 bits[i] 为 0 的时候，那么他就只能是向后移动一位了。所以算法如下：
```java
int len=0;
while(int len =0  && len< bits.length){
    len += bits[len] +1;
}
if(len == bits.length){
    return true;
}
```

## 解法二：
从后遍历，找出第一个为 0 的，如果第一个为0的是数组的倒数第二位的话，那么就是 true
```java
int len =bits.length-2;
while(len >0 && bits[len] >0 ){
    len --;
}
if(len == bits.length -2){
    return true
}
```

以上便是这两种解法了，只不过当时还未想到。
