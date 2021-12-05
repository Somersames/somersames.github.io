---
title: Spring中AOP的探索与实践(二)之Mongo多数据源切换
date: 2019-03-13 00:01:27
tags: [SpringBoot,Mongo]
categories: [Java,SpringBoot]
---
在之前的一片文章中介绍了使用AOP的方式来实现Redis的多数据源切换。而今天这一篇则是主要讲述`Mongo`的多数据源切换。

使用AOP来实现Mongo的数据源切换与Redis的AOP切换相同，不同之处是需要替换`MongoRepository`里面的`MongoOperations`,从而实现多数据源的切换


## 代码示例

配置类，读取Mongo的配置
```java
@Configuration
public class MongoMultiProperties {

    private static final Logger LOGGER = LoggerFactory.getLogger(MongoMultiProperties.class);

    @Bean(name="mongodb1")
    @Primary
    @ConfigurationProperties(prefix = "spring.data.mongodb.db1")
    public MongoProperties db1Properties(){
        LOGGER.info("正在初始化db1");
        return new MongoProperties();
    }

    @Bean(name = "mongodb2")
    @ConfigurationProperties(prefix = "spring.data.mongodb.db2")
    public MongoProperties db2Properties(){
        LOGGER.info("正在初始化db2");
        return new MongoProperties();
    }

}
```
配置`MongoRepository`
> 以下是为了演示，所以配置了两个MongoRepository，实际上使用了AOP的方式实现的多数据源，只需要配置一个默认的MongoRepository即可。


```java
@Configuration
@EnableMongoRepositories(mongoTemplateRef = "mongoDB2")
public class DB2Template {

    @Autowired
    @Qualifier("mongodb2")
    private MongoProperties mongoProperties;

    @Bean("mongoDB2")
    public MongoTemplate db2Template(){
        return new MongoTemplate(db2Factory(mongoProperties));
    }

    @Bean
    public MongoDbFactory db2Factory(MongoProperties mongoProperties){
        return new SimpleMongoDbFactory(new MongoClient(mongoProperties.getHost(),mongoProperties.getPort()),mongoProperties.getDatabase());
    }

}
```

```java
@Repository
public interface DB2Repository extends MongoRepository<MongoDB2,String>{
}

```

省略`DB1Template`的配置，基本上都是差不多的


## 一般使用
上述的配置如果都OK的话，则可以直接使用`@Autowired`注解使用。
```java
@Service
public class MongoService {
    @Autowired
    DB2Repository db2Repository;
    
    @Autowired
    DB1Repository db1Repository;

    public void mongoUpdate(){
        db2Repository.save(new MongoDB2());
    }
}
```

但是使用AOP的方式的话，切`Service`还是`Repository`是需要选择的，首先因为在业务使用中，肯定是包含许多的`Service`的，如果以后需要再添加其他的`Service`，还需要添加切点，比较麻烦。

如果是切`Repository`的话，那么这就好办了，直接配置一个主Repository，然后切这个主Repository，这样就可以将Service和AOP进行解耦。从而在Service里面，可以随意使用其他的数据源，例如:Mysql数据源，Redis数据源等。更加灵活

## 切面写法

```java
@Slf4j
@Component
@Aspect
@Order(-1)
public class MongoAspect implements ApplicationContextAware {

    private ApplicationContext applicationContext;


    @Around("execution(* com.somersames.config.mongo.db2.DB2Repository.*(..))")
    public Object doSwitch(ProceedingJoinPoint joinPoint) throws Throwable {
        return aopTest1(joinPoint);
//        aopTest2(joinPoint);
    }


    private Object aopTest1(ProceedingJoinPoint joinPoint) throws Throwable {
        Field methodInvocationField = joinPoint.getClass().getDeclaredField("methodInvocation");
        methodInvocationField.setAccessible(true);
        ReflectiveMethodInvocation o = (ReflectiveMethodInvocation) methodInvocationField.get(joinPoint);
        Field targetField = o.getClass().getDeclaredField("target");
        targetField.setAccessible(true);
        Object target = targetField.get(o);
        Field modifiersField = Field.class.getDeclaredField("modifiers");
        modifiersField.setAccessible(true);
        Object singletonTarget = AopProxyUtils.getSingletonTarget(target);
        Field mongoOperationsField = singletonTarget.getClass().getDeclaredField("mongoOperations");
        mongoOperationsField.setAccessible(true);
        //需要移除final修饰的变量
        modifiersField.setInt(mongoOperationsField,mongoOperationsField.getModifiers()&~Modifier.FINAL);
        mongoOperationsField.set(singletonTarget, applicationContext.getBean("mongoDB1"));
        return joinPoint.proceed();
    }


    private Object aopTest2(ProceedingJoinPoint joinPoint) throws Throwable {
        Field methodInvocationField = joinPoint.getClass().getDeclaredField("methodInvocation");
        methodInvocationField.setAccessible(true);
        ReflectiveMethodInvocation o = (ReflectiveMethodInvocation) methodInvocationField.get(joinPoint);
        Field h = o.getProxy().getClass().getSuperclass().getDeclaredField("h");
        h.setAccessible(true);
        AopProxy aopProxy = (AopProxy) h.get(o.getProxy());
        Field advised = aopProxy.getClass().getDeclaredField("advised");
        advised.setAccessible(true);
        Object o2 = advised.get(aopProxy);
        if (o2 instanceof Advised) {
            Object o1 = ((Advised) o2).getTargetSource().getTarget();
            Object o3 = AopProxyUtils.getSingletonTarget(o1);
            System.out.println(o3);
            Field mongoOperationsField = o3.getClass().getDeclaredField("mongoOperations");
            mongoOperationsField.setAccessible(true);
            Field modifiersField = Field.class.getDeclaredField("modifiers");
            modifiersField.setAccessible(true);
            //需要移除final修饰的变量
            modifiersField.setInt(mongoOperationsField, mongoOperationsField.getModifiers() & ~Modifier.FINAL);
            mongoOperationsField.set(o3, applicationContext.getBean("mongoDB1"));
        }
        return joinPoint.proceed();
    }

    @Override
    public void setApplicationContext(org.springframework.context.ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}
```

在上述的代码中，提供了两种的AOP的写法，但是最终都是获取`mongoOperations`，然后通过`applicationContext`来替换。

对比AOP的Redis写法，这里可以看到在Spring中的`AOP`实现，最起码使用`JDK动态代理`和`Cglib`。所以在本文中，使用的是
> Field h = o.getProxy().getClass().getSuperclass().getDeclaredField("h");

这个就是获取JDK动态代理的对象

至此，mongo的两种代理方式9最初版的代码编写完毕，后续可能需要对代码进行优化，从而避免每一次修改`application.yml`都需要手动添加`Repository`

完整代码可以访问[https://github.com/Somersames/Multi-Resource](https://github.com/Somersames/Multi-Resource)