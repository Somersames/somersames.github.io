---
title: netty的一些概念
date: 2018-05-31 15:10:19
tags: [Java,Netty]
categories: [第三方组件,Netty]
---

这里面的部分概念参考了《Apress JavaI.O . NIO and NIO2》
## Buffer
NIO的一些操作基础就是Buffer

## Channels
它的具体作用是帮助 DMA 快速的从硬盘上获取和写入数据

## Selector
选择器，目的是在异步模式中可以通过一个线程来实现哪些IO操作已经完成了。

