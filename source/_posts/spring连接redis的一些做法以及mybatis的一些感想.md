---
title: spring连接redis的一些做法以及mybatis的一些感想
date: 2018-03-27 20:38:19
tags: [spring,mybatis,Redis]
categories: [Java,Spring]
---

## Redis启动

首先开启redis服务，windows的redis下载在github [windows的redis下载地址](https://github.com/MicrosoftArchive/redis/releases),然后解压出来最后开启那个redis-server。
启动之后显示如下图：
![](redis.PNG)


## spring配置：
### 配置
在Spring中也可以通过配置文件`redis.properties`配置，但是由于在这个项目中的配置文件已经太多了，所以选择使用类的方式进行配置:
```
@Configuration
public class RedisConfig {
    @Bean
    JedisConnectionFactory jedisConnectionFactory()
    {
        JedisConnectionFactory connectionFactory = new JedisConnectionFactory();
        connectionFactory.setHostName("127.0.0.1");
        connectionFactory.setPort(6379);
        return new JedisConnectionFactory();
    }

    @Bean
    @Autowired
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory)
    {
        RedisTemplate<String, Object> template = new RedisTemplate<String, Object>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        return template;
    }
}
```
最后在使用的时候自动装配就可以了。
```
@Autowired
      private RedisTemplate redisTemplate;
```
### 单元测试
编写单元测试：
```
@ContextConfiguration("classpath:ApplicationContext.xml")
@RunWith(SpringJUnit4ClassRunner.class)
public class TestRedis  extends AbstractJUnit4SpringContextTests {

      @Autowired
      private RedisTemplate redisTemplate;
      @Test
      public void redisTest() throws UnsupportedEncodingException {
          redisTemplate.opsForValue().set("zhangsan", "book1");
          if (redisTemplate.hasKey("abc")){
              System.out.println("abc已经存入redis");
          }
      }
}
```

然后会发现数据已经添加到了这个redis服务器中了。




## Mybatis的一些知识点：
`SqlSesssionFactoryBuilder`:这是一个工厂，主要的作用是创建一些SqlSessioFactory，对于SqlSessionFactory来说，它的职责就是创建每一个SqlSession，然后这个SqlSession的作用则是连接数据库，类似于传统的JDBC的Connection，


SqlSesssionFactoryBuilder这个类主要是创建SqlSessioFactory,在这个类里面可以看到许多方法返回的是一个SqlSessionFactory，其中有一个方法是解析配置文件的方法：
```

public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
        SqlSessionFactory var5;
        try {
            XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
            var5 = this.build(parser.parse());
        } catch (Exception var14) {
            throw ExceptionFactory.wrapException("Error building SqlSession.", var14);
        } finally {
            ErrorContext.instance().reset();

            try {
                reader.close();
            } catch (IOException var13) {
                ;
            }

        }

        return var5;
    }

```
这里的第一个参数是一个流，这个流是将一个properties或者xml文件读取之后供mybatis使用的。
而对于SqlSessionFactory来讲，这是一个接口，其主要的接口方法如下：
```
public interface SqlSessionFactory {
    SqlSession openSession();

    SqlSession openSession(boolean var1);

    SqlSession openSession(Connection var1);

    SqlSession openSession(TransactionIsolationLevel var1);

    SqlSession openSession(ExecutorType var1);

    SqlSession openSession(ExecutorType var1, boolean var2);

    SqlSession openSession(ExecutorType var1, TransactionIsolationLevel var2);

    SqlSession openSession(ExecutorType var1, Connection var2);

    Configuration getConfiguration();
}
```
也就是说这个Factory主要负责的是Session的开启来查询数据库

### 注意的坑：
在mybatyis里面的话若使用对象传参的话需要注意`#{XXX}=YYY`，这里的YYY的话是需要和POJO相对应的，否则会提示` there is no getter for property named`