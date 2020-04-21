---
title: SpringBoot2.x是怎样只初始化LettuceConnectionFactory的呢
date: 2020-01-05 15:22:19
tags: [SpringBoot]
categories: SpringBoot
---
## SpringBoot1.5使用Redis和2.x的区别
在 SpringBoot1.5 的版本的时候，如果要创建一个 RedisTemplate 的话，那么可以直接使用如下代码:
```java
@Bean
public RedisTemplate<String,Object> redisTemplate(JedisConnectionFactory jedisConnectionFactory){
        RedisTemplate<String,Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(jedisConnectionFactory);
        // 添加序列化代码
        return redisTemplate；
}
```
然后在业务类中直接通过 `@Autowired` 注解来调用 redisTemplate，但是如果将 SpringBoot1.5 升级到 2.0 之后，你会发现这样写的话，SpringBoot 启动的时候会报错。报错内容如下：
```java
The following candidates were found but could not be injected:
- Bean method 'redisConnectionFactory' in 'JedisConnectionConfiguration' not loaded because @ConditionalOnBean (types: org.springframework.data.redis.connection.RedisConnectionFactory; SearchStrategy: all) found beans of type 'org.springframework.data.redis.connection.RedisConnectionFactory' redisConnectionFactory
    
Consider revisiting the entries above or defining a bean of type 'org.springframework.data.redis.connection.jedis.JedisConnectionFactory' in your configuration.

```

这个提示说明了 `redisConnectionFactory` 没有被加载到，这个时候先去官方文档上看下 changelog，然后在看下是不是有什么变化。
[SpringBoot1.x升级到2.X的简介](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.0-Migration-Guide#redis)
```java
Redis
Lettuce is now used instead of Jedis as the Redis driver when you use spring-boot-starter-data-redis. If you are using higher level Spring Data constructs you should find that the change is transparent.

We still support Jedis. Switch dependencies if you prefer Jedis by excluding io.lettuce:lettuce-core and adding redis.clients:jedis instead.

Connection pooling is optional and, if you are using it, you now need to add commons-pool2 yourself as Lettuce, contrary to Jedis, does not bring it transitively.
```
也就是说 SpringBoot2.x 已经将 LettuceConnectionFactory 作为官方的 Redis 连接工具了，那么尝试着将 `LettuceConnectionFactory` 作为那个 redisTemplate 的入参，最后调整如下：
```java
@Bean
public RedisTemplate<String,Object> redisTemplate(LettuceConnectionFactory lettuceConnectionFactory){
    RedisTemplate<String,Object> redisTemplate = new RedisTemplate<>();
    redisTemplate.setConnectionFactory(lettuceConnectionFactory);
    // 序列化代码
    return redisTemplate;
}
```
于是启动就不再报错了。而且也可以正常的使用。但是如果你还是想使用 Jedis 的话，直接 exclued `io.lettuce:lettuce-core` 即可。

## 为什么在SpringBoot2.x中两个RedisConnectionFactory只会加载一个
### SpringBoot2.x中加载的加载机制redisConnectionFactory的实现
首先 Redis 的一些配置都是依赖于 `RedisAutoConfiguration`，这个类是在 `spring-boot-autoconfigure` 里面，首先看下这个类的大体结构：
```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RedisOperations.class)
@EnableConfigurationProperties(RedisProperties.class)
@Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class })
public class RedisAutoConfiguration {

	@Bean
	@ConditionalOnMissingBean(name = "redisTemplate")
	public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory)
			throws UnknownHostException {
		RedisTemplate<Object, Object> template = new RedisTemplate<>();
		template.setConnectionFactory(redisConnectionFactory);
		return template;
	}

	@Bean
	@ConditionalOnMissingBean
	public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory redisConnectionFactory)
			throws UnknownHostException {
		StringRedisTemplate template = new StringRedisTemplate();
		template.setConnectionFactory(redisConnectionFactory);
		return template;
	}

}
```
首先这个类上面有四个注解，那么这四个注解的作用如下：
1. @Configuration(proxyBeanMethods = false)，声明这是一个配置类，同时也说明了不要让 SpringBoot 为这个类生成代理类
2. @ConditionalOnClass(RedisOperations.class)，当 classPath 中出现了 `RedisOperations` 这个类之后，该类才会加载成一个Bean
3. @EnableConfigurationProperties(RedisProperties.class)，将配置类 `RedisProperties` 加载进来，由于
4. @Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class })

在上述的几个注解里面，最重要的是 `@Import` 注解，这个注解可以允许我们在

TODO

## LettuceConnectionConfiguration 和 JedisConnectionConfiguration
当查看这两个类里面的方法的时候，会发现在创建 `redisConnectionFactory` 这个Bean的时候，都会有一个 @ConditionalOnMissingBean(RedisConnectionFactory.class) 注解。
```java
@Bean
@ConditionalOnMissingBean(RedisConnectionFactory.class)
LettuceConnectionFactory redisConnectionFactory(
        ObjectProvider<LettuceClientConfigurationBuilderCustomizer> builderCustomizers,
        ClientResources clientResources) throws UnknownHostException {
    LettuceClientConfiguration clientConfig = getLettuceClientConfiguration(builderCustomizers, clientResources,
            getProperties().getLettuce().getPool());
    return createLettuceConnectionFactory(clientConfig);
}

@Bean
@ConditionalOnMissingBean(RedisConnectionFactory.class)
JedisConnectionFactory redisConnectionFactory(
        ObjectProvider<JedisClientConfigurationBuilderCustomizer> builderCustomizers) throws UnknownHostException {
    return createJedisConnectionFactory(builderCustomizers);
}
```
很明显，这两个注解都是当 `RedisConnectionFactory.class` 这个 Bean 不存在的时候才会创建，而且在 `RedisAutoConfiguration` 中，首先 import 的是 `LettuceConnectionConfiguration`，所以最后才会导致在 SpringBoot2.x 的时候，默认加载的 `redisConnectionFactory` 是 `LettuceConnectionFactory`。


