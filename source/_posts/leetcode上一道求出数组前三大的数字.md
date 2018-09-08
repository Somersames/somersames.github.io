---
title: leetcode上一道求出数组前三大的数字
date: 2018-07-25 00:17:05
tags: [Leetcode,算法]
categories: Leetcode
---

题目如下:
```
Given scores of N athletes, find their relative ranks and the people with the top three highest scores, who will be awarded medals: "Gold Medal", "Silver Medal" and "Bronze Medal".

Input: [5, 4, 3, 2, 1]
Output: ["Gold Medal", "Silver Medal", "Bronze Medal", "4", "5"]
Explanation: The first three athletes got the top three highest scores, so they got "Gold Medal", "Silver Medal" and "Bronze Medal". 
For the left two athletes, you just need to output their relative ranks according to their scores.
```

也就是一个乱序的数组，然后将数组中的前三大的数组换成指定的字符串。其中有一个解法是比较有新意的，其思路如下：

另外再开辟一个数组，然后数组的长度是原数组中数字最大的那个值，那么在进行遍历的时候，新建的数组中最后三位肯定是最大的，然后依次将其原索引找出来即可。

具体的代码如下:
```java
 public String[] findRelativeRanks(int[] nums) {
        String[] s =new String[nums.length];
        int max =0;
        for(int i : nums){
            if(i> max){
                max= i;
            }
        }
        int[] index =new int[max+1];
        for(int i =0 ;i< nums.length ;i++){
            index[nums[i]] =i+1;  //在这里加一是因为如果不加一那么第一个索引的数组之在index里面会是0，不好区分。
        }

        int place=1;
        for(int i =index.length-1 ;i>=0 ;i--){
            if(index[i] !=0 ){
                if(place ==1 ){
                    s[index[i] -1]="Gold Medal";
                }else if(place ==2){
                    s[index[i] -1]="Silver Medal";
                }else if(place ==3){
                    s[index[i] -1]="Bronze Medal";
                }else{
                    s[index[i] -1]=String.valueOf(place);
                }
                place++;
            }

        }
        return s;
    }
```