---
title: Rabbitmq深度学习一
date: 2018-11-07 00:00:46
tags: [Rabbitmq]
categories: [消息队列,Rabbitmq]
---
## rabbitmq 深度学习一
### 简介
rabbitmq是目前使用最多的一个消息中间件，配合微服务的使用可以使业务模块化，便于之后的维护。同时使用rabbitmq可以将许多业务异步化，提高系统的性能。

### 队列的创建原则
但是在使用rabbitmq 的时候需要注意点就是到底是`生产者`来建立队列和交换机还是由`消费者`来建立交换机和队列。一般的情况是由消费者来建立队列，但是假如消费者挂掉了，导致生产者发出的消息被交换机路由到了一个不存在的队列，那么此时 rabbitmq会忽略该条消息。所以对于重要的消息。即不允许该消息丢失，那么此时最好是由生产者和消费者一起船创建一个队列。

在rabbit里面，假设生产者和消费者同时创建一个队列，如果队列的各项参数都相同的话，rabbitmq是不会有问题的，假设生产者和消费者同时创建一个队列，但是后创建的和前面创建的参数不同，那么此时rabbitmq就会报错。

#### 特殊的队列：死信队列
死信队列适用于当某一个队列的消息被拒绝的时候，或队列的消息大于最大TTL时以及队列大于最大值。此时该消息会被路由到死信交换器。
其一般的使用方式是建立一个死信交换机和一个死信队列，然后监听死信队列即可。

#### 代码示例：

##### 生产者端建立队列
```java
@Configuration
@ConfigurationProperties(ignoreUnknownFields = false, prefix = "somersames.rabbitmq")
@Data
public class RabbitmqConfig {

    private String laGouFailQueue = "laGouFailQueue";
    private String laGouQueue = "laGouQueue";
    private String laGouFailExchange = "laGouFailExchange";
    private String laGouFailExchangeRoutingKey = "laGouFailExchangeRoutingKey";
    public static final String DEAD_LETTER_EXCHANGE = "x-dead-letter-exchange";

    public static final String DEAD_LETTER_ROUTING_KEY = "x-dead-letter-routing-key";
    @Bean
    public Queue laGouFailQueue() {
        return new Queue(laGouFailQueue);
    }
    @Bean
    public DirectExchange laGouFailExchange() {
        return new DirectExchange(laGouFailExchange);
    }
    @Bean
    public Binding bindingLagouFailExchange(Queue laGouFailQueue, DirectExchange laGouFailExchange) {
        return BindingBuilder.bind(laGouFailQueue).to(laGouFailExchange).with(laGouFailExchangeRoutingKey);
    }
    @Bean
    public Queue laGouQueue() {
        Map<String, Object> map = new HashMap<>();
        map.put(DEAD_LETTER_EXCHANGE, laGouFailExchange);//设置死信交换机
        map.put(DEAD_LETTER_ROUTING_KEY, laGouFailExchangeRoutingKey);//设置死信routingKey
        return new Queue(laGouQueue, true, false, false, map);
    }
}

```

##### 消费者端建立队列
```java
@Configuration
public class RabbitConfig {

    private String laGouQueue = "laGouQueue";

    private String laGouExchange = "laGouExchange";

    private String laGouFailQueue = "laGouFailQueue";
    private String laGouFailExchange = "laGouFailExchange";
    private String laGouFailExchangeRoutingKey = "laGouExchangeRoutingKey";


    public static final String DEAD_LETTER_EXCHANGE = "x-dead-letter-exchange";

    public static final String DEAD_LETTER_ROUTING_KEY = "x-dead-letter-routing-key";

    @Bean
    public Queue laGouQueue() {
        Map<String, Object> map = new HashMap<>();
        map.put(DEAD_LETTER_EXCHANGE, laGouFailExchange);//设置死信交换机
        map.put(DEAD_LETTER_ROUTING_KEY, laGouFailExchangeRoutingKey);//设置死信routingKey
        return new Queue(laGouQueue, true, false, false, map);
    }
    @Bean
    public Queue laGouFailQueue() {
        return new Queue(laGouFailQueue);
    }
    @Bean
    public DirectExchange laGouFailExchange() {
        return new DirectExchange(laGouFailExchange);
    }


    @Bean
    public SimpleMessageListenerContainer laGouListenerContainer(
            ConnectionFactory connectionFactory,
            LagouRabbitMqListener lagouRabbitMqListener,
            Queue laGouQueue) {
        SimpleMessageListenerContainer container = new SimpleMessageListenerContainer();
        container.setMessageListener(lagouRabbitMqListener);
        container.setQueues(laGouQueue);
        container.setConnectionFactory(connectionFactory);
        container.setConcurrentConsumers(10);
        container.setMaxConcurrentConsumers(20);
        container.setAcknowledgeMode(AcknowledgeMode.MANUAL);
        return container;
    }
}

@Service
public class LagouRabbitMqListener implements ChannelAwareMessageListener {
    @Override
    public void onMessage(Message message, Channel channel) throws Exception {
        byte[] bytes = message.getBody();
        System.out.println(new String(bytes));
        channel.basicAck(message.getMessageProperties().getDeliveryTag(),false);
    }
}

```

### rabbitmq的路由方式
一般来讲rabbitmq的路由方式分为三种，一种是`fanout`，即该交换机会将此条消息推送到机绑定在该交换所有队列之中。

`topic`:该路由器可以支持通配符来进行消息的匹配，
```java
@Bean
    public Binding bindingPublicFundFailExchange(Queue queue, DirectExchange directExchange) {
        return BindingBuilder.bind(queue).to(directExchange).with("*.key");
    }
```
`#`可以匹配多个关键字
`*`只可以匹配一个关键字

`direct`:该路由器会精确匹配队列的key，从而路由器会将指定的消息发送到相应的队列
```java
@Bean
    public Binding bindingPublicFundFailExchange(Queue queue, DirectExchange directExchange) {
        return BindingBuilder.bind(queue).to(directExchange).with("error.key");
    }
```

topic是`fanout`和`diret`居中的一种模式，当`topic`为`#`的时候就类似于fanout，当`topic`的设置不带有`*` 和 `#` 的时候，就是一个`directe` 了。


### vhost多租户模式
在一个企业中，肯定不会是一个业务系统会使用rabbitmq，那么假设每一个业务系统都搭建一个自己的rabbitmq服务，此时就会造成极大的浪费。最好的解决办法是搭建一个集团一起使用的rabbbitmq集群，然后通过rabbitmq的vhost来进行一个隔离。

`vhost`模式类似于docker环境，一个vhost就是一个docker镜像，每一个vhost里面的交换机和队列都是互相隔离的。在rabbitmq中，默认的vhost就是一个`\`，
