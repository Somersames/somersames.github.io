---
title: springboot多数据源-sqlSessionFactory
date: 2020-03-19 19:57:04
tags: [SpringBoot]
categories: [Java,SpringBoot]
---
在SpringBoot中，动态的切换数据源的方式有两种，一种是通过`AbstractRoutingDataSource`来通过注解实现，另一种则是通过配置不同的`SqlSessionFactory`来读取不同文件夹的mapper，从而实现多数据源。
代码如下：
> DataSourceOneConfig


```java
@Configuration
@MapperScan(basePackages = {"xyz.somersames.dao.one"} ,sqlSessionTemplateRef = "dataSourceOneSqlSessionTemplate")
public class DataSourceOneConfig {

    @Primary
    @Bean(name = "dataSourceOneT")
    // 这里后面加一个T是防止Spring出现skiped mapperFactoryBean 错误，导致无法注入
    @ConfigurationProperties(prefix = "spring.datasource.one")
    public DataSource getDataSource(){
        return new DruidDataSource();
    }

    @Primary
    @Bean(name = "dataSourceOneSqlSessionFactory")
    public SqlSessionFactory setSqlSessionFactory(@Qualifier("dataSourceOneT") DataSource dataSource) throws Exception {
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        bean.setDataSource(dataSource);
        bean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:mapper/one/*.xml"));
        return bean.getObject();
    }

    @Primary
    @Bean(name = "dataSourceOneTransactionManager")
    public DataSourceTransactionManager setTransactionManager(@Qualifier("dataSourceOneT") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

    @Primary
    @Bean(name = "dataSourceOneSqlSessionTemplate")
    public SqlSessionTemplate setSqlSessionTemplate(@Qualifier("dataSourceOneSqlSessionFactory") SqlSessionFactory sqlSessionFactory) throws Exception {
        return new SqlSessionTemplate(sqlSessionFactory);
    }
}

```

>DataSourceTwoConfig


```java
@Configuration
@MapperScan(basePackages = {"xyz.somersames.dao.two"} ,sqlSessionTemplateRef = "dataSourceTwoSqlSessionTemplate")
public class DataSourceTwoConfig {

    @Bean(name = "dataSourceTwoT")
    // 这里后面加一个T是防止Spring出现skiped mapperFactoryBean 错误，导致无法注入
    @ConfigurationProperties(prefix = "spring.datasource.two")
    public DataSource getDataSource(){
        return new DruidDataSource();
    }

    @Bean(name = "dataSourceTwoSqlSessionFactory")
    public SqlSessionFactory setSqlSessionFactory(@Qualifier("dataSourceTwoT") DataSource dataSource) throws Exception {
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        bean.setDataSource(dataSource);
        bean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:mapper/two/*.xml"));
        return bean.getObject();
    }

    @Bean(name = "dataSourceTwoTransactionManager")
    public DataSourceTransactionManager setTransactionManager(@Qualifier("dataSourceTwoT") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

    @Bean(name = "dataSourceTwoSqlSessionTemplate")
    public SqlSessionTemplate setSqlSessionTemplate(@Qualifier("dataSourceTwoSqlSessionFactory") SqlSessionFactory sqlSessionFactory) throws Exception {
        return new SqlSessionTemplate(sqlSessionFactory);
    }
}

```
当添加了这两个配置以后，dataSourceOneT 会扫描 `mapper/one` 下面的 xml 文件，而 dataSourceTwoT 会扫描 `mapper/two` 下面的 xml 文件。
然后添加配置文件
> application.yml


```yml
server:
  port: 8091
spring:
  datasource:
    one:
      username: root
      password: 123456
      url: jdbc:mysql://localhost:3306/test?useSSL=false&serverTimezone=Asia/Shanghai
      type: com.alibaba.druid.pool.DruidDataSource
    two:
      username: root
      password: 123456
      url: jdbc:mysql://localhost:3306/test_lock?useSSL=false&serverTimezone=Asia/Shanghai
      type: com.alibaba.druid.pool.DruidDataSource
```
在这里顺带说一句，SqlSessionFactory 负责打开 SqlSession，一个SqlSession代表的就是一次回话，但是如果你在方法上加了一个 `@Transactional` 注解，那么这个方法里面的所有数据库操作都会认为是一个SqlSession