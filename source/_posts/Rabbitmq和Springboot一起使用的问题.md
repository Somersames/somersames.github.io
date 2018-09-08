---
title: Rabbitmq和Springboot一起使用的问题
date: 2018-08-04 00:42:51
tags: [rabbitmq,spring]
categories: Spring
---
刚刚准备在Springboot中使用rabbitmq来实现日志的收集，在使用中出现了一些问题，所以在此记录下。

## 消费者接收不到消息
这个问题出现的问题，一个是没有建立一个交换机，导致生产者在推送消息的时候使用的API是`this.amqpTemplate.convertAndSend(contet);`,直接将内容推出去，