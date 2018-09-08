---
title: mybatis和mysql得相关记录
date: 2018-04-10 23:30:41
tags: [web后端,mybatis]
categories: Java
---
首先对于Mybatis来说，如果是直接复制mysql里面的语句粘贴到mybatis的mapper文件里面去的话很容易导致user读取出错，假设在mysql中`select * from user XX`，若直接把这条sql语句复制到mapper文件中的话会导致user会成为mapper文件中的关键字