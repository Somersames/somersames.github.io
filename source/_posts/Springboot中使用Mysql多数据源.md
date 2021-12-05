---
title: Springboot中使用Mysql多数据源
date: 2019-01-06 20:55:12
tags: [Springboot]
categories: [Java,SpringBoot]
---
随着业务的发展，很可能需要在一个项目里面同时使用多个数据源。

大致看了网上的多数据源Demo，发现无非有两种：

> 一种是自己封装多个`JdbcTemplate`，然后调用对应的数据库就使用对应的`JdbcTemplate`

> 一种是通过注解的方式来实现，在需要切换数据源的方法上添加一个自己封装的注解便可以完成切换。

考虑了一下以后的扩展性和通用性，便决定采用基于注解的多数据源方式

## 分析
看了下官网的介绍，大致了解了在Spring中使用多数据源的一个关键类是`AbstractRoutingDataSource`。

先看下这个类的结构
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E8%B7%AF%E7%94%B1%E7%B1%BB%E5%9B%BE.png)

在这个类里面，只需要关注一下几个变量或者方法即可。
> 1. private Map<Object, Object> targetDataSources; 
> 2. private Object defaultTargetDataSource;
> 3. protected Object determineCurrentLookupKey()

### targetDataSources
这个变量是一个存储数据源的Map，其实一般在使用的时候，更加像是`Map<String, DataSource>`这样的，其中key表示的是这个数据源的名称，而value则是表示这个DataSource的信息(例如url，username等)

### defaultTargetDataSource
该变量表示的是一个默认的数据源，非空，必须设置

### determineCurrentLookupKey
该方法返回的一个`targetDataSources`里面的键，用于选择某一个数据源。其中多数据源的切换就是控制该方法的返回值来实现。
> 该方法返回的是`targetDataSources`里面的键，从而`HikariPool`可以直接切换数据源

## 思路
首先是通过读取配置文件，将其转为DataSource，然后再将dataSource存入`targetDataSources`

### 代码
配置文件
```yml
spring:
  application:
    name: multi-resource
  datasource:
    mysql1:
      driver-class-name: com.mysql.cj.jdbc.Driver
      jdbc-url: jdbc:mysql://localhost:3306/test?useUnicode=true&characterEncoding=UTF-8&serverTimezone=UTC
      username: root
      password: 123456
    mysql2:
      driver-class-name: com.mysql.cj.jdbc.Driver
      jdbc-url: jdbc:mysql://localhost:3306/cloud?useUnicode=true&characterEncoding=UTF-8&serverTimezone=UTC
      username: root
      password: 123456
  aop:
    auto: true
    proxy-target-class: true
```


获取配置文件信息
```java
@Slf4j
@Configuration
public class MysqlMultiProperties {

    private static final Logger LOGGER = LoggerFactory.getLogger(MysqlMultiProperties.class);

    @Bean("mysql1")
    @ConfigurationProperties("spring.datasource.mysql1")
    public DataSource mysql1Source(){
        log.info("正在初始化Mysql_DB1");
        return DataSourceBuilder.create().build();
    }

    @Bean("mysql2")
    @ConfigurationProperties("spring.datasource.mysql2")
    public DataSource mysql2Source(){
        log.info("正在初始化Mysql_DB2");
        return DataSourceBuilder.create().build();
    }
}
```


实现`AbstractRoutingDataSource `
```java
@Slf4j
public class DataSourceRouter extends AbstractRoutingDataSource {


    private static final ThreadLocal<String> contextHolder = new ThreadLocal<>();

    @Override
    protected Object determineCurrentLookupKey() {
        return getDataSource();
    }

    public static void setDataSource(String dataSource) {
        contextHolder.set(dataSource);
    }

    public static String getDataSource() {
        return contextHolder.get();
    }

}
```

实例化Bean
```java
@Configuration
public class MysqlConfig {

    @Autowired
    @Qualifier("mysql1")
    DataSource mysql1;


    @Autowired
    @Qualifier("mysql2")
    DataSource mysql2;

    @Bean
    @Primary
    public DataSourceRouter generateRouter(){
        DataSourceRouter router =new DataSourceRouter();
        Map<Object,Object> targetMap = new HashMap<Object, Object>();
        targetMap.put("mysql1",mysql1);
        targetMap.put("mysql2",mysql2);
        router.setTargetDataSources(targetMap);
        router.setDefaultTargetDataSource(mysql1);
        router.afterPropertiesSet();
        return router;
    }

}
```

### AOP切面
**在使用切面的时候遇到了一些坑，这个有空再说**

新建一个注解
```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface UseDataSource {
    String name();
}

```

新建一个切面类
```java
@Slf4j
@Component
@Aspect
@Order(-1)
public class MysqlSourceAspect {

    @Before("@annotation(useDataSource)")
    public void changeMysqlSource(UseDataSource useDataSource){
        DataSourceRouter.setDataSource(useDataSource.name());
    }
}
```


至此大部分功能都已经实现了，在Springboot的启动类上添加或修改如下注解
```java
@SpringBootApplication(exclude={DataSourceAutoConfiguration.class})
@MapperScan(basePackages = "com.somersames.dao")
@Import({MysqlConfig.class})
public class ServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceApplication.class);
    }
}
```

## 使用

只需要在切换数据源的地方添加`@UseDataSource`注解即可。

项目地址：https://github.com/Somersames/Multi-Resource


## 总结
这次在编写AOP部分的时候需要了一点小坑，有空会整理出来