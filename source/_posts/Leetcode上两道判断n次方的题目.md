---
title: leetcode上两道判断n次方的题目
date: 2018-09-26 23:41:40
tags: [算法]
categories: [算法,leetcode]
---
这两道题目都是判断一个数字是不是2(第一题)，3(第二题)的n次方，在做第一题的时候思路基本上和标准解法想法相同，但是在做第二题的时候，看到了许多比较有创意的解法，所以记录下

## 判断一个数是不是2的n次方
### 解法一
这个解法也就是我第一次就想到的一个解法，就是做 `&` 运算，因为一个数字若是2的n次方，那么很明显就是这个数字的2进制肯定只会有一个`1`，例如：
32=100000 ,64 =1000000。所以只需要判断 n 与 n-1 做一个`&` 运算就可以知道了。
```java
public boolean isPowerOfTwo(int n) {
        if( n <1){
            return false;
        }
        return (n & ( n-1)) ==0;
    }
```
### 解法二
在Java里面。Int的最大值是`2^31 - 1`到 `-2^31`次，所以很明显，只需要让 n 与 `2^30` 次做一个 `&` 运算即可。
```java
public boolean isPowerOfTwo(int n) {
        if( n<1){
            return false;
        }
        return (1073741824 % n) == 0;
    }
```


## 判断一个数是不是3的n次方
### 标准解法
```java
 public boolean isPowerOfThree(int n) {
        if(n ==0){
        return false;
        }
       while(n % 3 ==0){
           n = n/3;
       }
        return n ==1;
    }
```

### 解法二
在Java里面int的最大值是`2^30`，那么用3的最大值就可以是`3^19`,所以解法二为：
```java
public boolean isPowerOfThree(int n) {
        if( n<1){
            return false;
        }
        return (1162261467 % n) == 0;
    }
```

### 解法三

由数学公式: n= 3^1,可以得到
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20180927002429.png)

所以会有以下代码：
```java
 public boolean isPowerOfThree(int n) {
        return (Math.log10(n) / Math.log10(3)) % 1 == 0;
    }
```
