---
title: mybatis中多条语句插入和主键返回
date: 2018-04-09 23:42:11
tags: [mybatis,web后端]
categories: Java
---
在Mybatis的使用中，有时候会出现需要一对多的场景，尤其是在插入的过程中，即假设存在A，B两表。A表对B表是一对多的关系，在插入数据的过程中需要先插入A表，通过A表返回的主键然后再进行B表的查询，这个时候一般有两种操作。
> 方法一：首先获取A表的主键，然后通过for循环进行B表的插入
方法二：使用mybatis的`foreach`进行多条语句的插入

在这里的话主要是记录下第二种方法，即通过mybatis的`foreach`来实现多条语句的插入:


## 建立数据库表
```
mysql> select * from clazz;
+----------+------------+
| CLAZZ_ID | CLAZZ_NAME |
+----------+------------+
|        1 | 一年级     |
+----------+------------+
1 row in set (0.00 sec)


mysql> select * from student;
+----------+--------+----------+
| CLAZZ_ID | STU_ID | STU_NAME |
+----------+--------+----------+
|        1 |    101 | a        |
|        1 |    102 | b        |
+----------+--------+----------+
2 rows in set (0.00 sec)
```
在数据库建立的两张表会发现班级表和学生表是一对多的关系，此时如果一个班级需要插入许多个学生的话，此时就要使用mybatis的`foreach`进行多条语句的插入了



## 占位符
在这里得话需要插入多条语句得时候不能使用`#`，而是应该使用`$`。在这里为什么是这个美元的符号，暂时得猜测是`$`解析得为静态得，也就是说mybatis会讲集合里面的值直接填充进来，类似于在ApplicationContext.xml中动态的加载`jdbc.properties`，需要使用`$`符号。
新建Dao层类：
```java
public interface clazzStudent {
    Integer insertToStudent(List<Map<String,Object>> map);
}
```
然后建立与之对应的mapper文件：
```xml
<insert id="insertToStudent" >
        insert into student(CLAZZ_ID,STU_ID,STU_NAME) values 
        <foreach collection="list" item="item"  index="index" separator=",">
            (${item.get("clazz")},${item.get("stu")},${item.get("name")})
        </foreach>
    </insert>
```
最后建立测试类：
```java
public static void main(String args[]) {
        Logger logger = null;
        logger = Logger.getLogger(MybatisExample.class.getName());
        logger.setLevel(Level.DEBUG);
        SqlSession sqlSession = null;
        try {
            sqlSession = study.mybatis.MybatisUtil.getSqlSessionFActory().openSession();
            clazzStudent clazz = sqlSession.getMapper(clazzStudent.class);
            List<Map<String,Object>> list =new ArrayList<Map<String, Object>>();
            for(int i=0 ;i<3;i++){
                Map<String,Object> itemmap =new HashMap<String, Object>();
                itemmap.put("clazz",1);
                itemmap.put("stu",i);
                itemmap.put("name","'adsads'");
                list.add(itemmap);
            }
            int n=clazz.insertToStudent(list);
            System.out.println(n);
            sqlSession.commit();
        } finally {
            sqlSession.close();
        }
    }

```
在这里需要注意得是` <foreach collection="list" item="item"  index="index" separator=",">`在这个里面的话`collection="list"`这里的list其实不需要和main方法中得list名称对应，也就是说在Java得main方法中将` List<Map<String,Object>> list =new ArrayList<Map<String, Object>>()`这个list改成list1也可以插入多条语句，这里参数`list`与`collection`中得名称无关。
### separator
分隔符，作用是foreach下面的语句执行完了之后之后用什么进行分割。而`item`则代表得是里面得那个map，
### index
index代表的是索引，目前暂时没用到，所以就写一个索引


## 利用Java对象进行迭代
修改代码之后也可以利用Java对象进行传参
```java
  Logger logger = null;
        logger = Logger.getLogger(MybatisExample.class.getName());
        logger.setLevel(Level.DEBUG);
        SqlSession sqlSession = null;
        try {
            sqlSession = study.mybatis.MybatisUtil.getSqlSessionFActory().openSession();
            clazzStudent clazz = sqlSession.getMapper(clazzStudent.class);
            List<Student> list1 =new ArrayList<Student>();
            for(int i=0 ;i<3;i++){
             Student student =new Student();
             student.setCLAZZ_ID(1);
             student.setSTU_ID(i);
             student.setSTU_NAME("测试");
             list1.add(student);
            }
            int n=clazz.insertToStudentPojo(list1);
            System.out.println(n);
            sqlSession.commit();
        } finally {
            sqlSession.close();
        }
```
但是使用对象传参的话需要注意得是在mapper文件中使用`#`符号，修改如下：
```xml
<insert id="insertToStudentPojo">
        insert into student(CLAZZ_ID,STU_ID,STU_NAME) values
        <foreach collection="list" item="item"  index="index" separator=",">
            (#{item.CLAZZ_ID},#{item.STU_ID},#{item.STU_NAME})
        </foreach>
    </insert>
```
## 用List传值：
新建Dao层方法
```java
Integer insertToStudentList(List<String> list);
```
新建mapper方法：
```xml
 <insert id="insertToStudentList">
        insert into student(STU_NAME) values
        <foreach collection="list" item="item"  index="index" separator=",">
            (${item})
        </foreach>
    </insert>
```
测试方法：
```java
public static  void insertList(){
        Logger logger = null;
        logger = Logger.getLogger(MybatisExample.class.getName());
        logger.setLevel(Level.DEBUG);
        SqlSession sqlSession = null;
        try {
            sqlSession = study.mybatis.MybatisUtil.getSqlSessionFActory().openSession();
            clazzStudent clazz = sqlSession.getMapper(clazzStudent.class);
            List<String> list1 =new ArrayList<String>();
            for(int i=0 ;i<3;i++){
             list1.add(String.valueOf(i));
            }
            int n=clazz.insertToStudentList(list1);
            System.out.println(n);
            sqlSession.commit();
        } finally {
            sqlSession.close();
        }
    }
```

## 区别
在这里需要对Java对象传参使用得`#`和map传参得`$`进行一下区分。可以看下打印日志：
java对象传参使用`#`传参得日志：
```log
DEBUG [main] - ==>  Preparing: insert into student(CLAZZ_ID,STU_ID,STU_NAME) values (?,?,?) , (?,?,?) , (?,?,?) 
DEBUG [main] - ==> Parameters: 1(Integer), 0(Integer), 测试(String), 1(Integer), 1(Integer), 测试(String), 1(Integer), 2(Integer), 测试(String)
DEBUG [main] - <==    Updates: 3
```

java使用List<Map>进行传参:
```log
DEBUG [main] - ==>  Preparing: insert into student(CLAZZ_ID,STU_ID,STU_NAME) values (1,0,'adsads') , (1,1,'adsads') , (1,2,'adsads') 
DEBUG [main] - ==> Parameters: 
DEBUG [main] - <==    Updates: 3
3
```
可以看到打印出来的日志发现`$`和`#`在解析上得区别，`$`这个是先解析，生成了一个sql之后再执行，而`#`则是一个占位符，是在后期的时候动态加入的。

## 主键返回
mybatis的主键返回目前仅仅了解通过对象来返回。需要在mapper文件中加入一行配置即可：
```xml
<insert id="insertToStudentPojo">
        <selectKey keyProperty="CLAZZ_ID" order="AFTER" resultType="java.lang.Integer">
            select LAST_INSERT_ID()
        </selectKey>
        insert into student(CLAZZ_ID,STU_ID,STU_NAME) values
        <foreach collection="list" item="item"  index="index" separator=",">
            (#{item.CLAZZ_ID},#{item.STU_ID},#{item.STU_NAME})
        </foreach>
    </insert>
```
`keyProperty="CLAZZ_ID"`表示的是需要返回的字段的值
```java
 public static void pojo(){
        Logger logger = null;
        logger = Logger.getLogger(MybatisExample.class.getName());
        logger.setLevel(Level.DEBUG);
        SqlSession sqlSession = null;
        try {
            sqlSession = study.mybatis.MybatisUtil.getSqlSessionFActory().openSession();
            clazzStudent clazz = sqlSession.getMapper(clazzStudent.class);
            List<Student> list1 =new ArrayList<Student>();
            for(int i=0 ;i<3;i++){
             Student student =new Student();
             student.setCLAZZ_ID(1);
             student.setSTU_ID(i);
             student.setSTU_NAME("测试");
             list1.add(student);
            }
            int n=clazz.insertToStudentPojo(list1);
            System.out.println(list1.get(2).getCLAZZ_ID());
            System.out.println(n);
            sqlSession.commit();
        } finally {
            sqlSession.close();
        }
    }
```
需要注意的是` System.out.println(list1.get(2).getCLAZZ_ID());`因为是多条语句的插入，肯定是需要返回的最后一条插入后的主键，所以通过该方法便可以得到插入后的主键

## 总结

目前就暂时这么多了，源码以后再了解了。