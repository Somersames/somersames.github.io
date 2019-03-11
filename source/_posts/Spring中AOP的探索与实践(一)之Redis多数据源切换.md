---
title: Spring中AOP的探索与实践(一)之Redis多数据源切换
date: 2019-03-12 00:16:00
tags: [Springboot,Redis]
categories: Springboot
---
一般在项目的使用过程中，有时候为了减轻数据库的压力，从而将一部分数据缓存至Redis，但是随着业务量的增多。我们所需要的Redis服务器也会越来越多，就算不需要多个Redis数据源，那么在一个redis里面，切换不同的DB也是很麻烦的一件事情。


## 非AOP的一般的多数据源操作
在Redis的多数据源使用中，一般的方法是从配置文件中读取多个`RedisProperties`，读取到配置文件之后，将`RedisProperties`配置到`RedisTemplate`，然后每次使用的时候就通过不同的`Template`来调用Redis服务器。示例如下:

### 代码示例

#### Redis的配置类
```java
@Configuration@Order(1)
@Slf4j
public class RedisConfig {


    @Bean(name = "redis1")
    @Primary
    JedisConnectionFactory jedisConnectionFactory1(){
        JedisConnectionFactory jedisConnectionFactory =new JedisConnectionFactory();
        jedisConnectionFactory.setHostName("127.0.0.1");
        jedisConnectionFactory.setPort(6379);
        jedisConnectionFactory.setPassword("123456");
        jedisConnectionFactory.setDatabase(0);
        return jedisConnectionFactory;
    }
    
    @Bean(name = "redis2")
    @ConfigurationProperties("spring.redis.db1")
    JedisConnectionFactory jedisConnectionFactory2(){
        JedisConnectionFactory jedisConnectionFactory =new JedisConnectionFactory();
        jedisConnectionFactory.setHostName("127.0.0.1");
        jedisConnectionFactory.setPort(6379);
        jedisConnectionFactory.setPassword("123456");
        jedisConnectionFactory.setDatabase(2);
        return jedisConnectionFactory;
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> template = new RedisTemplate<String, Object>();
        template.setConnectionFactory(jedisConnectionFactory1());
        return template;
    }
    @Bean
    public RedisTemplate<String, Object> redisTemplate2() {
        RedisTemplate<String, Object> template = new RedisTemplate<String, Object>();
        template.setConnectionFactory(jedisConnectionFactory2());
        return template;
    }
}
```
在上面的配置类里面，分别生成了两个`JedisConnectionFactory`和两个`RedisTemplate`，那么在使用的时候直接通过注解`@Autowired`装配两个`RedisTemplate`即可。

#### 使用示例
```java
@Service
public class RedisCommonService {
    @Autowired
    RedisTemplate<String,Object> redisTemplate ;

    @Autowired
    RedisTemplate<String,Object> redisTemplate2 ;

    public void redis1Save(String key,String value){
        redisTemplate.opsForValue().set(key,value);
    }

    public void redis1Save2(String key,String value) {
        redisTemplate2.opsForValue().set(key, value);
    }
}
```
#### 调用
```java
RestController
@RequestMapping("/")
public class RedisCommonController {

    @Autowired
    RedisCommonService redisCommonService;

    @RequestMapping(value = "/redis1",method = RequestMethod.GET)
    public void redis1(){
        redisCommonService.redis1Save("1","2");
    }


    @RequestMapping(value = "/redis2",method = RequestMethod.GET)
    public void redis2(){
        redisCommonService.redis1Save2("2","3");
    }
}

```


然后在Redis的服务器上可以看到
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/Redis%E5%A4%9A%E6%95%B0%E6%8D%AE%E6%BA%90.png)

可以看到两个数据分别写入到了不同的Db中


## 通过AOP的调用
通过AOP方法调用的基础是需要获取`RedisTemplate`里面的`JedisConnectionFactory`
切面代码如下：

```java
@Around("execution(* com.somersames.service.redis.RedisService.*(..))")
    public void as(ProceedingJoinPoint joinPoint) throws Throwable {
        Field methodInvocationField = joinPoint.getClass().getDeclaredField("methodInvocation");
        System.out.println(AopUtils.isAopProxy(joinPoint.getTarget()));
        methodInvocationField.setAccessible(true);
        ReflectiveMethodInvocation o = (ReflectiveMethodInvocation) methodInvocationField.get(joinPoint);
        Field h = o.getProxy().getClass().getDeclaredField("CGLIB$CALLBACK_0");
        h.setAccessible(true);
        Object dynamicAdvisedInterceptor = h.get(o.getProxy());
        Field advised = dynamicAdvisedInterceptor.getClass().getDeclaredField("advised");
        advised.setAccessible(true);

        Object target = ((AdvisedSupport)advised.get(dynamicAdvisedInterceptor)).getTargetSource().getTarget();
        Field re = target.getClass().getDeclaredField("redisTemplate");
        re.setAccessible(true);
        Object re2= re.get(target);

        Field d =  re2.getClass().getSuperclass().getDeclaredField("connectionFactory");
        d.setAccessible(true);
        Object[] objs = joinPoint.getArgs();
        if(objs != null && objs.length !=0){
            re.set(target,applicationContext.getBean((String) objs[0]));
        }

        joinPoint.proceed();
    }
```
#### RedisServer
```java
@Service
public class RedisService {

    @Autowired
    RedisTemplate<String,Object> redisTemplate;


    public void aopRedis(String reditTemplate){
        redisTemplate.opsForValue().set("a","a");
    }

}
```
上述的代码也很简单，就是获取`RedisServer`的`aopRedis`方法的第一个参数，然后通过AOP将其替换为指定的Redis连接。测试如下:
```java
@RestController
@RequestMapping("/")
public class AopController {


    @Autowired
    RedisService redisService;
    
    @RequestMapping(value = "/redis",method = RequestMethod.GET)
    public void testCurd1(){
        redisService.aopRedis("redisTemplate2");
    }
}

```

在`AopController`里面，我想通过`redisTemplate2`来执行`aopRedis`方法。
但是在`RedisService`里面，我们又是配置的是Redis连接数据源1，那么如何
> RedisTemplate<String,Object> redisTemplate;

这个时候，我们可以通过切面，直接替换RedisTemplate的连接，从而获取指定的Redis连接，测试如下：
启动服务。
访问`http://localhost:8080/redis`。


在不开启切面的情况下，可以看到直接访问的是`0`号库，而开启切面之后，在调用`RedisService`的时候，由于切面将RedisTemplate的`connectionFactory`替换为2号库，所以访问结果如下:
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/RedisAo.png)



本篇文章只是简单的介绍了下AOP的使用，下面几篇可能会基于这篇文章做一些AOP补充和增加一些其他功能。例如：添加Redis的AOP的自动切换，同时添加多个Redis数据源的自动注入，不再手动写Bean。
然后会可能基于Mongo的多数据源来讲解AOP的不同代理获取方式，和一般通用的获取方式