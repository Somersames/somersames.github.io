---
title: leetcode的一道字符串题目：最长不重复的子字符串长度
date: 2018-03-14 20:58:32
tags: [算法]
categories: [算法,leetcode]
---

# 找出最长的不重复的子字符串

首先一拿到这个题目的时候想到利用set集合来存储子字符串，当发现含有重复的字符的时候就将这个set清零。这个做法有一个明显的缺点就是当子字符串中的某一个字符和后面的重复的话，那么后面不行同的也会被清楚：例如`dfvfab`，在这个里面`vfab`应该是最短的子字符串，但是这时候这个方法会输出错误的值。

## 思路：
自己想不到解法之后就参考了Solution2，这个方法想了下也挺好的，遂记录下：
该方法是设置两个游标，一个在前，一个在后(定义为i,j)，当前面的一个发现在set集合中含有重复的字符，那此时停止i,然后j自增，当发现在set集合中已经将重复的那个字符剔除之后，此时在i自增
```
public int lengthOfLongestSubstring(String s) {
        //错误的思路，以为存入set就可以
        // if (s == null || s.length() == 0){
        //     return 0;
        // }
        // Set<Character> set =new HashSet<>();
        // int max=0;
        // for (int i =0 ;i< s.length() ;i++){
        //     char c = s.charAt(i);
        //     if (set.contains(c)){
        //         set.clear();
        //     }
        //     set.add(c);
        //     if (set.size() > max){
        //         max = set.size();
        //     }
        // }
        // return max;
        if(s == null || s.length() ==0){
            return 0;
        }
        Set<Character> set =new HashSet<>();
        int i=0,j=0,result=0,n=s.length();
        while(i < n && j <n){
            if(!set.contains(s.charAt(j))){
                set.add(s.charAt(j++));
                result=Math.max(result,j-i);
            }else{
                set.remove(s.charAt(i++));
            }
        }
        return result;
    }
```

## 与求最长的子字符串（朴素求解法）相比较：
    朴素求解法的一般步骤是已经给出了子字符串，然后判断是否是另一个字符串的子字符串。那么此时既然已知子字符串的话，其步骤就是通过子字符串依次比较



## 与判断链表成环相比较：
    判断链表成环也有一个快慢指针的方法，但是在那个快慢指针中，快慢指针都是同时在变化的
