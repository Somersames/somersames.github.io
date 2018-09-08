---
title: Leetcode上两道比较好的BFS和DFS题目
date: 2018-06-14 23:38:47
tags: [算法,Leetcode]
categories: Leetcode
---
首先是Leetcode上两道比较好的一个题目，分别如下：
* `https://leetcode.com/problems/letter-case-permutation/description/` Letter Case Permutation
* `https://leetcode.com/problems/max-area-of-island/description/` Max Area of Island

关于字符串的那一题便是将一个字符串里面的任意一个字符进行大小写的变换，此题有两种解法，一种是 BFS 按照字符窜中的字符遍历，将其变成大小写，然后存入栈中，最后便每一次向后迭代，然后再存入即可。另一种则是 DFS ，通过一种不断递归的方式来进行大小写的变换的，和爬楼梯的那个算法极其类似
# 字符串

## 字符串的BFS
伪代码：
```java
Stack <- stack;
for i -> S.length(){
    if(i is char){
        stack.pop() -> y;
        y[i] <- Upper
        stack.push(y[i]);
        y[i] <- Lower
        stack.push(y[i]);
    }
}
```
在这个代码的写法中，采取的是广度有限遍历，即在一个平面上展开，而不是深入到这个字符的

## 字符串的DFS
伪代码:
```java
List <- list ;
char[] c <- S.toCharArray();
fun change(list,index){
if( index = S.length()){
    list.add(S);
    return;
}
if(c[index] is not char){
    change(list,index+1);
}
char[] temp <- S.toCharArray();
temp[index] <- UpperCase;
change(list,index+1);

temp[index] <- Lower;
change(list,index+1);
}
```


## 关于这两个算法的差异

一个是从广度借助额外的存储空间来进行大小写的变换，而另一个则是通过递归将这个字符串从尾到头的进行大小写的变化。


# 最大岛屿面积的DFS
题意就是从给定的二维数组中找出数字为 1 的，并且要求它们之间不能有间隙，所以这一题是比较适合 DFS 的解法，其类似于上楼梯的那一道题目，上楼梯就是一个递归，把每一次的步数罗列出来。

## 面积题目的DFS
伪代码:
```java
boolean[][] <- x
fun getMaxArea(int[][] a, x,int xIndex, int yIndex ){
    if( xIndex < 0 ,yINdex<0 , xINdex<a.length, yINdex<a[0].length){
        return 0;
    }
    x <- true;
    return  1 + getMaxArea(a,x,xIndex+1,yIndex)+getMaxArea(a,x,xIndex,yIndex+1) + getMaxArea(a,x,xIndex-1,yIndex) getMaxArea(a,x,xIndex,yIndex-1)
}
```