---
title: 使用Mq实现延时队列
date: 2019-03-04 00:30:53
tags: [Rabbitmq]
categories: Rabbitmq
---
在实际的应用开发中，下游系统并不需要立即处理上游系统的mq，但是又不可能将消息阻塞在上有系统中。且这两个系统之间又没有接口提供出来。这个时候就需要通过Mq的死信队列来实现一个延时效果

## 简介
由于Mq的发送方不支持延迟发送(**目前的新版本可以使用插件来支持，但是可能由于公司的其他限制，导致无法升级**)，这时候就需要使用Mq的死信队列来实现延时队列

一般情况下，Mq的发送流程如下：

![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/Mq%E4%B8%80%E8%88%AC%E6%B5%81%E7%A8%8B.png)


## Mq的死信队列
在Mq的使用中，死信队列用于处理一下三种情况：
> 1. The message is negatively acknowledged by a consumer using basic.reject or basic.nack with requeue parameter  set to false
> 2. The message expires due to TTL; or
> 3. The message is dropped because its queue exceeded a length limit

上述三种情况分别是
1. 消费者拒绝了该Mq，同时消费者也设置该消息不重新入队。
2. 消息过期，即无人消费
3. 队列设置了长度，同时队列已满

由于Mq不支持消息的延迟发送，即消息一经投递，就会马上入队，到达交换机。然后根据交换机的属性，进行投递。

这样就带来了一个问题，如果发送方需要消费者等待一定的时间才能进行消费。如果不经由死信队列，那么只能在发送方做等待。

例如使用`Thread.sleep()`来实现，由于`Sleep`方法是阻塞的，所以这样做又会影响到性能。又或者通过定时任务来实现，但是定时任务每一次又要去取出该发送的Mq，然后再发出去，这样就会非常的影响到效率。


## 死信队列实现延迟队列
通过Rabbitmq的死信转发转发规则2，便可以实现一个延时队列。具体流程如下：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/Mqexpire.png)

这里实现的关键是讲消费队列设置为`DLK`,`TTL`,`DLX`，具体如下：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/Mqmanage.png)

在上图里面，可以看到`Tutorials`是一个消费队列，同时已经设置了它的死信转发规则。而`dTutorials`是一个死信队列，这个队列就是用于存放死信队列



## 代码如下：
```java
        ConnectionFactory factory = new ConnectionFactory();

        factory.setHost(HOST);
        factory.setUsername(USER);
        factory.setPassword(PASSWORD);
        factory.setPort(5672);
        factory.setVirtualHost("/");

        // 声明一个连接
        connection = factory.newConnection();

        channel = connection.createChannel();
        
        
        
        // 在这里分别新建两个Exchange，一个是死信的Exchange，一个是消费者的Exchange
        channel.exchangeDeclare(exchangeName,"direct",true,false,null);
        channel.exchangeDeclare(dExchangeName,"direct",true,false,null);

    
        byte[] bytes  =messgae.getBytes();
        //声明一个队列 - 持久化，同时设置死信的转发Exchange和Queue。以及消息的过期时间
        Map<String,Object> args = new HashMap<String, Object>();
        args.put("x-dead-letter-exchange",dExchangeName);
        args.put("x-dead-letter-routing-key",routingKey);
        args.put("x-message-ttl",2000);
        channel.queueDeclare(queueName, true, false, false, args);
        channel.queueDeclare(dQueeueName, true, false, false, null);

        //设置通道预取计数
        channel.basicQos(1);

        //将消息队列绑定到Exchange
        channel.queueBind(queueName, exchangeName, routingKey);
        channel.queueBind(dQueeueName, dExchangeName, routingKey);
        channel.basicPublish(exchangeName, routingKey, null, bytes);
    }
```

生产者代码：

```java
ConnectionFactory factory = new ConnectionFactory();

        factory.setHost(HOST);
        factory.setUsername(USER);
        factory.setPassword(PASSWORD);
        factory.setPort(5672);
        factory.setVirtualHost("/");

        // 声明一个连接
        connection = factory.newConnection();

        // 声明消息通道
        channel = connection.createChannel();
        
        
        
        //在这里消费者直接消费死信队列即可
        channel.queueBind(dQueueName, dExchangeName, routingKey);
        DefaultConsumer defaultConsumer =new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String str = new String(body);
                System.out.println(str);
                System.out.println(new Date());
                if("测试".equals(str)){
                    channel.basicReject(envelope.getDeliveryTag(),false);
                }else{
                    channel.basicAck(envelope.getDeliveryTag(),false);
                }
            }
        };
        channel.basicConsume(dQueueName,defaultConsumer);
```
下图中可以看到，消费者总是在2秒钟之后收到了发送方发送的消息，这时一个延时队列就实现了

![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/GIF2.gif)



## 详细代码连接
具体的代码以上传至Github:https://github.com/Somersames/MqTutorials