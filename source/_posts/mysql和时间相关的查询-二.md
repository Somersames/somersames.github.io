---
title: MySql和时间相关的查询(二)
date: 2018-03-28 00:06:05
tags: [MySql]
categories: [数据库,MySql]
---

## DATE_ADD():
这个函数可以将日期往前加上规定的年，月或者日，从而方便统计，例如需要统计本月的某些数据的话，一般来讲肯定是只需要大于本月月初即可，但是为了考虑程序的健壮性的话肯定是需要再加一个限制条件，比如说小于下个月1号。那么就需要一个一个DATE_ADD()函数：
```
SELECT
  *
FROM
  table
WHERE time_columns > CONCAT(LEFT(NOW() - INTERVAL 0 MONTH, 7), '-01')
  AND time_columns < DATE_ADD(CURDATE()-DAY(CURDATE())+1,INTERVAL 1 MONTH);
```
这条sql便是求的本月的数据。同时在这里也对当前时间的函数进行一个对比：
```
mysql> SELECT CURDATE();
+------------+
| CURDATE()  |
+------------+
| 2018-03-28 |
+------------+
1 row in set (0.00 sec)

mysql> select now();
+---------------------+
| now()               |
+---------------------+
| 2018-03-28 15:07:15 |
+---------------------+
1 row in set (0.00 sec)

mysql> select CURTIME();
+-----------+
| CURTIME() |
+-----------+
| 15:07:39  |
+-----------+
1 row in set (0.00 sec)

```

顺便在这里再记录下mybatis中的别名的作用，在resultType中返回的通常是全路径包名，但是其实配置好别名的话其实是可以直接返回别名> 深入浅出mybatis技术原理与实现P17以及后面的mapper文件的resultType

另外就是mybaytis的`$`和`#`的区别，其实`$`这个在配置文件中是很经常使用的，例如:
```
<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">
        <!-- 基本属性 url、user、password -->
        <property name="driverClassName" value="com.mysql.jdbc.Driver"></property>
        <property name="url" value="${url}" />
        <property name="username" value="root" />
        <property name="password" value="${password}" />
</bean>
```
在这里就是用的`$`,而且在mybatis的官方文档里面也提到过了关于`#`和`$`的区别"

> String Substitution
  By default, using the #{} syntax will cause MyBatis to generate PreparedStatement properties and set the values safely against the PreparedStatement parameters (e.g. ?). While this is safer, faster and almost always preferred, sometimes you just want to directly inject an unmodified string into the SQL Statement. For example, for ORDER BY, you might use something like this:
  ORDER BY ${columnName}
  Here MyBatis won't modify or escape the string.
  NOTE It's not safe to accept input from a user and supply it to a statement unmodified in this way. This leads to potential SQL Injection attacks and therefore you should either disallow user input in these fields, or always perform your own escapes and checks.



也就是说在这里的话`$`是一个静态占位符,但是在mybatis的源文件中一直没找到关于解析`$`的类和方法，只找到了关于解析`#`的方法：
```
 public SqlSource parse(String originalSql, Class<?> parameterType, Map<String, Object> additionalParameters) {
        SqlSourceBuilder.ParameterMappingTokenHandler handler = new SqlSourceBuilder.ParameterMappingTokenHandler(this.configuration, parameterType, additionalParameters);
        GenericTokenParser parser = new GenericTokenParser("#{", "}", handler);
        String sql = parser.parse(originalSql);
        return new StaticSqlSource(this.configuration, sql, handler.getParameterMappings());
    }
```
关于这个`${}`的话在mybatis里面是会直接将我们所写的sql放入到预定的sql语句中，而不是像`#{}`那样会进行一个替换，所以会造成一个安全问题
