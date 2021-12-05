---
title: 记一次sql和时间段查询有关的语句
date: 2018-03-23 23:11:30
tags: [MySql]
categories: [数据库,MySql]
---
## 前言
由于自己有点需求需要在mysql中按照时间段进行查询，而自己又对这些不太了解，所以趁着这次机会将mysql和时间段相关的查询语句做一个记录：

## 常用函数：
#### DATE_SUB()函数
`DATE_SUB(date,INTERVAL expr type)`这个函数的date是一个时间表达式，一般取得是数据库中的一个字段。后面的`INTERVAL`一般来讲是不变的，`expr`一般是一个时间段，代表过去的，比如是30天，那么这里就是30，若是60，这里就是60，`type`则表示的是一个时间属性(可能表达的不是很准确)，如下：
```
MICROSECOND
SECOND
MINUTE
HOUR
DAY
WEEK
MONTH
QUARTER
YEAR
SECOND_MICROSECOND
MINUTE_MICROSECOND
MINUTE_SECOND
HOUR_MICROSECOND
HOUR_SECOND
HOUR_MINUTE
DAY_MICROSECOND
DAY_SECOND
DAY_MINUTE
DAY_HOUR
YEAR_MONTH

```

在这里我用`YEAR_MONTH` 和` MONTH` 同时写了一个sql`SELECT DATE_SUB('2015-09-30 11:19:00',INTERVAL2 MONTH)`在这个里面，将MONTH和YEAR_MONTH互换都是没问题的，这是目前暂时未发现有什么差异的。


#### FROM_UNIXTIME()函数
这是一个非常有用的函数，其作用是将一串数字或者一个时间戳转换成指定格式的一个日期，在这个里面的话第一个参数需要是一个UNIX时间戳,而第二个参数则是一个format的字符串，注意观察下面的三个例子的不同，第二个sql在后面做了一个+0的操作结果日期格式直接被转换成了一串数字。
```
mysql> SELECT FROM_UNIXTIME(1447430881);
        -> '2015-11-13 10:08:01'
mysql> SELECT FROM_UNIXTIME(1447430881) + 0;
        -> 20151113100801
mysql> SELECT FROM_UNIXTIME(UNIX_TIMESTAMP(),
    ->                      '%Y %D %M %h:%i:%s %x');
        -> '2015 13th November 10:08:01 2015'

```

#### CONCAT()函数
`CONCAT()`函数可以将一个字符串按照第二个参数，具体来说就是对应以下语句`SELECT LEFT('2018-01-26 11:50:09',7)`这条语句的执行结果结果是2018-01,正好是从头截取了7个字符。
参数：第一个是字符串，第二个是截取的长度。
```
mysql> SELECT LEFT('2018-01-26 11:50:09',7);
+-------------------------------+
| LEFT('2018-01-26 11:50:09',7) |
+-------------------------------+
| 2018-01                       |
+-------------------------------+
1 row in set (0.00 sec)

```
#### LEFT()函数
`LEFT()`函数可以将一个字符串作为mysql的语句来执行，类比到Python就是python中的`eval`函数，但是该函数若执行的字符串中包含一个`null`:注意这个null若是关键字则是会则会导致
```
mysql> select concat('Mysql',null);
+----------------------+
| concat('Mysql',null) |
+----------------------+
| NULL                 |
+----------------------+
1 row in set (0.00 sec)
```
#### sum()和Count()在统计1和0的区别
这两个函数的区别在一次使用统计的时候遇到了一点问题，例如一个字段是标识符，也就是一个字段的值除了0就是1，那么要统计为1的话一般来讲就是`case when xxx=1 then 1 else 0 end`这样就可以值选择为1的行，那么这里如果使用的是count的话，这里的这个规则是无法应用的，具体如下：
```

mysql> select * from testestatus;
+-----------+---------------+
| status_id | status_status |
+-----------+---------------+
|         1 |           b'1'|
|         2 |           b'0'|
|         3 |           b'1'|
|         4 |           b'1'|
+-----------+---------------+
4 rows in set (0.00 sec)

```
因为当时没有仔细的区分这两种用法的区别，所以才导致了统计错误。在这里，count只是一个统计，他只是统计应用规则之后的的符合数据，但是sum不同，sum配合`case when` 则可以选择出相匹配的一些数据
#### DATE_FORMAT()函数
这个函数可以将一个时间按照指定的格式输出，类似于Java中的dateformat函数，是可以将一个时间进行格式的。
```
mysql> select date_format(now(),'%Y');
+-------------------------+
| date_format(now(),'%Y') |
+-------------------------+
| 2018                    |
+-------------------------+
1 row in set (0.00 sec)
```

### 一些常用函数组合：
###### 查询本月的数据：
在这里是可以使用`concat`函数和`left`函数，首先可以使用`left`函数截取一个字符串长度。比如获取当月的话是可以通过left来获取然后再将1号拼接上去，最后用数据库的日期再减去1号就可以了。
```
select left(now(),7);
+---------------+
| left(now(),7) |
+---------------+
| 2018-03       |
+---------------+
1 row in set (0.00 sec)


mysql> select concat(left(now(),7),'-01');
+-----------------------------+
| concat(left(now(),7),'-01') |
+-----------------------------+
| 2018-03-01                  |
+-----------------------------+
1 row in set (0.00 sec)

```
最后本月的1号就被查询出来了，然后便可以进行数据操作了

###### 查询指定天数的数据
假设现在需要查询往期30天的数据，那么可以通过`date_sub`函数来进行查询：
```
select date_sub(now(),INTERVAL 30 DAY );
+----------------------------------+
| date_sub(now(),INTERVAL 30 DAY ) |
+----------------------------------+
| 2018-02-22 22:03:22              |
+----------------------------------+
1 row in set (0.00 sec)

```

## 总结：
目前关于时间方面的sql总结差不多就是这些了，其他的以后再进行补充。