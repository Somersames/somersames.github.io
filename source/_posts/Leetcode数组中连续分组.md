---
title: Leetcode数组中连续分组
date: 2019-09-08 23:19:58
tags: [leetcode]
categories: 算法
---
在Leetcode上有一道题目，如下：
> In a deck of cards, each card has an integer written on it.
  Return true if and only if you can choose X >= 2 such that it is possible to split the entire deck into 1 or more groups of cards, where:
  Each group has exactly X cards.
  All the cards in each group have the same integer.


这一题就是一个求最大公约数的题目，当任意两组的公约数为1的时候，那么此时就说明，他们的分组数量不相等就可以直接返回false了。
```java
public boolean hasGroupsSizeX(int[] deck) {
        int[] count = new int[10000];
        for(int val : deck){
            count[val]++;
        }

        int sum = -1;
        for(int val: deck){
            if(sum == -1){
                sum = count[val];
            } else {
                sum = getGcd(sum,count[val]);
            }
            if(sum == 1){
                return false;
            }
        }
        return true;

    }
```
