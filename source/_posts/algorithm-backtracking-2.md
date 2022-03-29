---
title: 算法系列「回溯」-排列、切割、子集
toc: true
date: 2022-03-06 01:22:49
tags: [算法]
categories: [算法,回溯]
---
# 回溯定义
按照维基百科上给到的定义：回溯是一种最优选择法，如果发现某一些选择不符合目标，那么就会进行退回操作

在了解回溯算法之前，首先就不得不提到一个概念「递归」，递归 和 回溯 在基本的逻辑上其实是差不多的，只不过回溯都含有撤销操作，而递归则是一条路走到底，两者都属于暴力搜索法。

最经典的题目就属于「八皇后」

# 递归
## 斐波拉契数
> 斐波那契数 （通常用 F(n) 表示）形成的序列称为 斐波那契数列 。该数列由 0 和 1 开始，后面的每一项数字都是前面两项数字的和。[509. 斐波那契数](https://leetcode-cn.com/problems/fibonacci-number/)

这一题就是典型的递归，其公式表达为 `F(x) = F(x - 1) + F(x - 2)`
```java
class Solution {
    public int fib(int n) {
        return add(n);
    }

    private int add(int n) {
        if(n == 0){
            return 0;
        }
        if(n == 1){
            return 1;
        }
        return add(n-1) + add(n-2);
    }
}
```

### 递归和循环的转换
每一次的递归都可以转换为「循环」，递归其实就是将上一次方法的计算值带入到下一次的计算当中，而循环其实也是有这个能力的，例如将上面的递归改成循环：
```java
class Solution {
    public int fib(int n) {
        return add(n);
    }
    
    private int add(int n) {
        if(n == 0){
            return 0;
        }
        if(n == 1){
            return 1;
        }
        int[] array = new int[n + 1];
        array[0] = 0;
        array[1] = 1;
        int total = 1;
        for(int i = 2; i <= n; i++){
            array[i] =  array[i-1] + array[i - 2];
        }
        return array[n];
    }
}
```
for 循环的版本还有很多，但是与本文的「回溯」没太多关系，感兴趣的可以去看力扣的官方题解，里面的方法比较多。

## 递归所不能解决的问题
例如一个数组[1,2,2,3]，需要算出该数组有多少个不重复的子集，这就是递归所无法解决的问题，类似的还有组合、切割等

# 回溯
回溯就是在递归的过程中不断判断当前条件与目标是否一致，如果不一致的话，那么就需要撤销当前的操作（俗称剪枝），回溯通常用来解决以下几个问题：
1. 排列
2. 切割
3. 子集
4. 组合
5. 棋盘问题

## 排列
> 百度百科的定义：从n个不同元素中取出m（m≤n）个元素，按照一定的顺序排成一列，叫做从n个元素中取出m个元素的一个排列(permutation)。特别地，当m=n时，这个排列被称作全 排列(all permutation)。
### [46. 全排列](https://leetcode-cn.com/problems/permutations/)
> 给定一个不含重复数字的数组 nums ，返回其 所有可能的全排列 。你可以 按任意顺序 返回答案。

标准模版一：
```java
class Solution {
    public List<List<Integer>> permute(int[] nums) {
        List<List<Integer>> result = new ArrayList<>();
        List<Integer> list = new ArrayList();
        for(int i : nums){
            list.add(i);
        }
        search(nums.length,list,result,0);
        return result;
    }

    private void search(int n,List<Integer> tempList,List<List<Integer>> result,int index) {
        if(index == n){
            result.add(new ArrayList(tempList));
            return;
        }
        for(int i = index; i < n; i++){
            Collections.swap(tempList,index,i);
            search(n,tempList,result,index + 1);
            Collections.swap(tempList,i,index);
        }
    }
}

```
### [47. 全排列 II](https://leetcode-cn.com/problems/permutations-ii/)
> 给定一个可包含重复数字的序列 nums ，按任意顺序 返回所有不重复的全排列。
```java
class Solution {

    List<List<Integer>> result = new ArrayList<>();

    public List<List<Integer>> permuteUnique(int[] nums) {
        Boolean[] visited = new Boolean[nums.length];
        Arrays.fill(visited,false);
        Arrays.sort(nums);
        search(new ArrayList<>(),visited,nums);
        return result;
    }

    private void search(List<Integer> list,Boolean[] visited,int[] nums) {
        if(list.size() == nums.length){
            result.add(new ArrayList<>(list));
            return;
        }
        for(int i = 0; i< nums.length; i++){
            if(visited[i]){
                continue;
            }
            if(i > 0 && nums[i] == nums[i - 1] && visited[i - 1] == false){
                continue;
            } 
            visited[i] = true;
            list.add(nums[i]);
            search(list,visited,nums);
            visited[i] = false;
            list.remove(list.size() - 1);
        }
    }
}
```

## 切割
### [698. 划分为k个相等的子集](https://leetcode-cn.com/problems/partition-to-k-equal-sum-subsets/)
> 给定一个整数数组  nums 和一个正整数 k，找出是否有可能把这个数组分成 k 个非空子集，其总和都相等。
```java
class Solution {
    int[] nums = new int[]{};
    public boolean canPartitionKSubsets(int[] nums, int k) {
        int sum = Arrays.stream(nums).reduce(0,Integer::sum);
        if(sum % k != 0){
            return false;
        }
        Arrays.sort(nums);
        this.nums = nums;
        return backtracking(k,sum / k, 0, 0,new boolean[nums.length]);
    }

    private boolean backtracking(int k,int target,int total,int currentIndex,boolean[] used) {
        if(k == 1){
            return true;
        }
        if(total == target){
            return backtracking(k - 1, target, 0, 0,used);
        }
        for(int i = currentIndex; i< nums.length; i++){
            if(used[i]){
                continue;
            }
            if(total + nums[i] > target){
                continue;
            }
            used[i] = true;
            if(backtracking(k,target,total + nums[i],i + 1,used)){
                return true;
            }
            used[i] = false;
        }
        return false;
    }
}
```
### [131. 分割回文串](https://leetcode-cn.com/problems/palindrome-partitioning/)

```java
class Solution {
    String S = "";
    List<List<String>> result = new ArrayList<>();
    public List<List<String>> partition(String s) {
        this.S = s;
        backtracking(0,new ArrayList<>(),s);
        return result;
    }

   private void backtracking(int curr,List<String> list,String s) {
        if(curr == s.length()){
            result.add(new ArrayList<>(list));
            return;
        }
        for(int i = curr; i < s.length(); i++){
            String tmp = s.substring(curr,i + 1);
            if(isPalindrome(tmp)){
                list.add(tmp);
                backtracking(i + 1,list,s);
                list.remove(list.size() - 1);
            }
        }
    }

    private boolean isPalindrome(String s){
        if(s == null || s.length() == 0){
            return false;
        }
        return s.equals(new StringBuilder(s).reverse().toString());
    }
}
```
## 子集
### [78. 子集](https://leetcode-cn.com/problems/subsets/)
> 给你一个整数数组 nums ，数组中的元素 互不相同 。返回该数组所有可能的子集（幂集）。解集 不能 包含重复的子集。你可以按 任意顺序 返回解集。
```java
class Solution {

List<List<Integer>> result = new ArrayList<>();

    public List<List<Integer>> subsets(int[] nums) {
        backtracking(0,nums,new ArrayList<>());
        return result;
    }

    private void backtracking(int current, int[] nums, List<Integer> list) {

        if(current == nums.length){
            result.add(new ArrayList<>(list));
            return;
        }
        backtracking(current + 1, nums,list);
        list.add(nums[current]);
        backtracking(current + 1, nums,list);
        list.remove(list.size() - 1);
    }
}
```
### [90. 子集 II](https://leetcode-cn.com/problems/subsets-ii/)
> 给你一个整数数组 nums ，其中可能包含重复元素，请你返回该数组所有可能的子集（幂集）。解集 不能 包含重复的子集。返回的解集中，子集可以按 任意顺序 排列。

**for 循环版本**
```java
class Solution {

    List<List<Integer>> result = new ArrayList<>();

    public List<List<Integer>> subsetsWithDup(int[] nums) {
        List<Integer> temp = new ArrayList<>();
        Arrays.sort(nums);
        result.add(temp);
        backtracking(0,nums,temp);
        return result;
    }

    private void backtracking(int index,int[] nums,List<Integer> list) {
        if(index == nums.length){
            return;
        }
        for(int i = index; i< nums.length; i++){
            if(i > index && nums[i] == nums[i - 1]){
                continue;
            }
            list.add(nums[i]);
            result.add(new ArrayList<>(list));
            backtracking(i + 1,nums,list);
            list.remove(list.size() - 1);
        }
    }
}
```
**递归版本**
```java
class Solution {

    List<List<Integer>> result = new ArrayList<>();

    public List<List<Integer>> subsetsWithDup(int[] nums) {
        List<Integer> temp = new ArrayList<>();
        Arrays.sort(nums);
        backtracking(0,nums,temp,new boolean[nums.length]);
        return result;
    }

     private void backtracking(int index,int[] nums,List<Integer> list,boolean[] used) {
        if(index == nums.length){
            result.add(new ArrayList<>(list));
            return;
        }
        backtracking(index + 1,nums,list,used);
        if(index > 0 && nums[index] == nums[index-1] && !used[index - 1]){
            return;
        }
        used[index] = true;
        list.add(nums[index]);
        backtracking(index + 1, nums, list, used);
        list.remove(list.size() - 1);
        used[index] = false;
    }
}
```
