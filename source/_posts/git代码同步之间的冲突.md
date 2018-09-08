---
title: git代码同步之间的冲突
date: 2018-04-08 22:47:20
tags: [工具]
categories: Git
---
在使用git的时候单独一人进行`push`和`pull`的时候是不会出现代码冲突的，但是当团队中有多人的时候进行协作的时候难免会造成代码间同步问题。
具体就是git pull的时候会提示线上代码会覆盖本地的代码。然后就不让pull，最后也不让push。查询了下解决办法：

## 方法一：
```shell
git stash
git pull 
git stash pop
```
但是这种方法pull下来的代码会导致IDEA识别不了。也就是java文件会直接不显示，最后是关闭IDEA然后再次打开才解决这个问题的。重新导入的时候选择maven



## 方法二：
这个方法主要应对的是push提交不上去，但是pull却又显示是线上最新版本
```linux
git reset --hard
git push
```