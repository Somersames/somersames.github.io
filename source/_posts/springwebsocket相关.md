---
title: springwebsocket相关
date: 2018-04-16 00:00:37
tags: [websocket,web后端]
---
在使用Spring来进行websocket通信的时候需要注意几点问题：


## localhost与127.0.0.1的区别
在使用`127.0.0.1`的时候是可以和服务器建立websocket连接的，但是如果想在websocket的intercept里面拦截request，并且获取session里面的用户的话。若在js脚本里卖弄使用`127.0.0.1`的话会会导致浏览器在发起请求的时候不会带上cookies，所以在这里获取的一定会是一个null，这里便需要注意，也就是说在浏览器中输入的是localhost那么在websocket中输入的也必须是localhost，否则会导致浏览器在进行websocket请求的时候不会带上cookie