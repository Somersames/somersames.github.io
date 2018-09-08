---
title: 'Mybatis中sql排序以及#和$的区别(二)'
date: 2018-04-12 23:30:16
tags: [mybatis,web后端]
categories: Java
---

## mybatis的orderby
在使用mybatis的时候，一般来讲是使用`#{}`这种方式来设置sql的参数，因为mybatis在解析sql的时候的时候对于使用`#{}`的参数首先会解析成`?`,然后再加入参数。具体可以看mybatis的解析日志：
```
DEBUG [main] - ==>  Preparing: select * from clazzentity where clazz_name = ? 
DEBUG [main] - ==> Parameters: 一年级(String)
```
但是在mybatis中如果需要使用groupby和orderby的话就需要注意不可以使用`#`了，因为使用`#`的话会导致解析出来的参数自动的带了一个引号,而使用`$`的话就会直接把参数带进去，所以在进行groupby的时候是需要使用`$`来进行参数的替换的。但是在使用${}这个的时候需要注意下。

# mybatis的多参数和单参数
### 单参数
mybatis的单参数一般来讲是会自动被识别的，`#`的话直接可以加参数名称，`$`的话则是使用`_paramter`来接收


### 多参数
#### param
mybatis中若需要使用多参数的话要么使用`@param`注解，要么使用集合或者对象来作为参数,其代码如下：
```java
  List<Map<String,Object>> queryStudnet2(@Param("CLAZZID") String clazzId ,@Param("name") String name);
```
而其配置文件则如下图所示：
```xml
<select id="queryStudnet2" resultType="java.util.HashMap" >
        select count(CLAZZ_ID) as 'id' , CLAZZ_ID from student GROUP by ${CLAZZID} order by ${name}
    </select>
```
在${}里面的参数全部是@Param,所以这是一种多参数传递方式，同时这种方式进行传参的话#和$都可以解析，

#### List
List的传参应该只是适用于foreach迭代,也就是需要动态的执行多条语句的时候才会使用List


#### Java对象
而Java对象传参的话在入参的地方需要加入Java对象的实体类，但是在查询需要返回结果的时候可以通过resultMap接收，不是一个map也可以是一个Java对象也可以接收，但是需要注意的是通过Java对象接收的话要java字段和sql的字段映射。一种就是已经说了的resultMap，另一种就是数据库返回字段设置别名然后让其返回的是Java对象的字段

#### Map
map则适用于多参数传递，类似于java对象
通过map传递参数的话是最常见的一种方式在传入`#{}`的动态参数，但是在使用map的方式的时候需要注意，通过map传值的话，用`${}`是接受不到的，会直接抛出异常
```
### Error querying database.  Cause: org.apache.ibatis.builder.BuilderException: Error evaluating expression '_paramter.clazz'. Cause: org.apache.ibatis.ognl.OgnlException: source is null for getProperty(null, "clazz")
### Cause: org.apache.ibatis.builder.BuilderException: Error evaluating expression '_paramter.clazz'. Cause: org.apache.ibatis.ognl.OgnlException: source is null for getProperty(null, "clazz")
```
所以map传值的话最好是传入需要`#{}`接受的参数

#### 索引传参
仅适用于#
```xml
 <select id="queryStudnet3" resultType="java.util.HashMap" >
        select count(CLAZZ_ID) as 'id' , CLAZZ_ID from student GROUP by #{arg0} order by #{arg1}
</select>
```
Java代码如下：
```java
   //Dao层代码
   List<Map<String,Object>> queryStudnet3(String clazzId ,String name);
   //实际方法
    List<Map<String,Object>> list =clazz.queryStudnet3("CLAZZ_ID","id");
```
通过这个方式传参的话需要注意mybatis的提示使用`arg0`还是`param`
# 异常汇总
```
 Cause: org.apache.ibatis.executor.ExecutorException: A query was run and no Result Maps were found for the Mapped Statement 'study.dao.clazzStudent.queryStudnet2'.  It's likely that neither a Result Type nor a Result Map was specified
```
这个表示若返回的是一个集合的话需要制定下返回的类型。其代码如下所示：
