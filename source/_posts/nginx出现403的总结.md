---
title: nginx出现403的总结
date: 2018-05-14 23:44:40
tags: [web后端,nginx]
categories: nginx
---
今天在使用nginx的时候访问首页总是提示403forbidden，经过各种查询之后，总结为如下几种原因：
1. 访问的资源权限不足，最好将nginx访问的资源权限修改为`755`或者`777`
2. SeLinux的设置为true，需要将其修改为false

如果出现首页的访问资源不是指定的目录的话，可以在`/etc/nginx/nginx.conf`中添加一条语句`root XXX`，XXX代表的是资源目录。
其他的暂时没发现什么问题