---
title: Rabbitmq深度学习二
date: 2018-11-10 00:22:34
tags: [rabbitmq]
categories: Rabbitmq
---
## 消息的接收和拒绝
当消费段接收到一个消息之后，会进行消费的处理。假如在业务中发现该消息是一个错误的消息，那么很显然业务方会直接拒绝此条消息。

### 拒绝消息

拒绝消息一般在消费端调用`reject`或者`neck`，这两个Api的作用分别是：一个可以批量消息，一个则只是每一次处理一条

### 接收消息

一般直接调用`basic.ack`即可

## 发送方的消息确认

### 消息的确认
既然rabbitmq作为一款成熟的消息中间件，那么自然也有完善的消息确认机制。消息确认机制分为生产者端和消费者端，生产者端的消息确认主要是用于确认消息是否成功的到达交换机以及其绑定的队列。

使用生产者端的消息确认需要实现`RabbitTemplate.ConfirmCallback`和`abbitTemplate.ConfirmCallback`，并且重写`confirm`方法，例如一下代码:
```java
@Service
public class MqConfirmCallback implements RabbitTemplate.ConfirmCallback {
    @Override
    public void confirm(CorrelationData correlationData, boolean b, String s) {
        if(b) {
            System.out.println("消息确认成功");
        }else{
            System.out.println("消息确认失败");
        }
    }
}
```


```java
@Service
public class MqReturnCallback implements RabbitTemplate.ConfirmCallback {
    @Override
    public void confirm(CorrelationData correlationData, boolean b, String s) {
        System.out.println("确认回调被出发");
    }
}
```
当重写完这两个方法之后还不行，另外还需要的是在`RabbitTemplate`设置其`setConfirmCallback`和`setReturnCallback`,然后即可使用。
```java
@Autowired
    MqConfirmCallback mqConfirmCallback;

    @Autowired
    MqReturnCallback mqReturnCallback;
    
    @PostConstruct
    public void init(){
        rabbitTemplate.setConfirmCallback(mqConfirmCallback);
        rabbitTemplate.setReturnCallback(mqReturnCallback);
    }
```

此时启动生产者但是却不启动消费者，然后发送一条消息，在控制台可以看到打印出来了`消息确认成功`,此时表明消息已经成功的到达了交换机，那么此时如果我将绑定在交换机上的所有队列删除呢?

#### 删除交换机以及队列来测试

此时消息因为到不了交换机，控制台会打印`确认回调被出发`和`消息确认成功`，至于此处明显是由于消息路由不到交换机，而导致消息被Return。那么为啥那么还会出发一次`消息确认成功`呢？。查看Rabbitmq的文档发现：
> For unroutable messages, the broker will issue a confirm once the exchange verifies a message won't route to any queue (returns an empty list of queues). If the message is also published as mandatory, the basic.return is sent to the client before basic.ack. The same is true for negative acknowledgements (basic.nack)

意思就是当消息不可路由的时候，则broker会在`basic.ack`之前发送一个`basic.return`。那么也就不奇怪了为什么这里会收到两条消息

#### 消费者拒绝测试

既然mq的发送方会有一个确认，那么如果消费者拒绝了此条消息，发送方还会收到提示吗？
可以看到在控制台只会打印`消息确认成功`，也就是说发送方只会确认此条消息到达了交换机，而且交换机可以路由到指定对的队列。


### 不可重复在一个message进行多次操作

假设在一个消息上进行了拒绝之后，然后再进行确认，此时`mq`会抛出一个异常：
```
Caused by: com.rabbitmq.client.ShutdownSignalException: channel error; reason: {#method<channel.close>(reply-code=406, reply-text=PRECONDITION_FAILED - unknown delivery tag 1, class-id=60, method-id=80)
```

### 不要将错误的消息重新归队

若一个消息已经明确知道会导致消费方异常，则不要将此条消息拒绝然后重新写到队列，否则会导致一个循环，从而阻塞后面的消息