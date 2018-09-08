---
title: 在mysql5.7版本中使用groupby所需要注意的
date: 2018-03-25 12:37:13
tags: mysql
categories: 数据库
---

## 什么是ONLY_FULL_GROUP_BY 模式
先看在mysql 5.7版本中的一个的group by，以下是这个数据库表：
```
mysql> select * from testgroupby;
+---------+-----------+------------+--------------+
| user_id | user_name | user_score | user_subject |
+---------+-----------+------------+--------------+
|       1 | 张三      |         99 | 语文         |
|       2 | 张三      |         90 | 数学         |
|       3 | 张三      |         80 | 英语         |
|       4 | 李四      |         99 | 语文         |
|       5 | 王五      |         85 | 语文         |
|       6 | 李四      |         91 | 数学         |
|       7 | 王五      |        100 | 英语         |
+---------+-----------+------------+--------------+
7 rows in set (0.00 sec)
```

这是一个学生成绩数据库表。那么在MySQL5.7版本中执行它的话是会出现一个error的。如下所示：

```
mysql> select * from testgroupby group by user_name;
ERROR 1055 (42000): Expression #1 of SELECT list is not in GROUP BY clause and contains nonaggregated column 'login.testgroupby.user_id' which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by

```
注意查看报错：`nonaggregated column 'login.testgroupby.user_id' which is not functionally dependent on columns`。这段话是什么意思呢？大意表示的是对于groupby的username，userid并没有函数依赖于它。在这张表中，可以由`user_id`推导出`user_name`。因为每一个`user_id`都是唯一的，若对`user_id`进行groupby 则它可以推导出任何一个学生姓名，分数以及学科。即每一个学生Id都是可以确定一行值得，所以MySQL可以返回数据。 那么反过来是不是每一个学生姓名都可以推导出唯一的`user_id`呢？ 在这张表中，每一个`user_name`都是不可以推导出唯一的user_id-也就是说对`user_name`分组之后会有多余的`user_id`-, 所以也可以认为`user_id`并没有函数依赖于`user_name`。其实在这里还可以这样想，因为对`user_name`分组之后，在每一个组里面都会有几个值，那么随之而来的`user_id ,user_score,user_subject`都不是一个确定的值，也就是说在一个分组里面，在这几列中会有多个值，那么mysql这是就会不知道到底该返回哪一行得，所以这个时候开启了而这个模式的mysql就会拒绝查询。

## 解决方法：
### 一：
那么如何才让它不报这个错误呢，在mysql的官方文档上面说的是:
> The query is valid if name is a primary key of t or is a unique NOT NULL column. In such cases, MySQL recognizes that the selected column is functionally dependent on a grouping column. For example, if name is a primary key, its value determines the value of address because each group has only one value of the primary key and thus only one row. As a result, there is no randomness in the choice of address value in a group and no need to reject the query


也就是说如果groupby的列是一个主键的话，mysql会识别出他的一个函数依赖。在这个表中，由于对`user_id`进行groupby，在分组之后mysql是可以进行查询的。
重新修改查询语句：
```
mysql> select * from testgroupby group by user_id;
+---------+-----------+------------+--------------+
| user_id | user_name | user_score | user_subject |
+---------+-----------+------------+--------------+
|       1 | 张三      |         99 | 语文         |
|       2 | 张三      |         90 | 数学         |
|       3 | 张三      |         80 | 英语         |
|       4 | 李四      |         99 | 语文         |
|       5 | 王五      |         85 | 语文         |
|       6 | 李四      |         91 | 数学         |
|       7 | 王五      |        100 | 英语         |
+---------+-----------+------------+--------------+
7 rows in set (0.00 sec)
```

也就是说如果当`groupby`后面的字段是是一个非空主键的时候，由于主键是一个表中的唯一标识符，不可以重复，所以MySQL可以正确的推断出每一个分组。

那如果还是需要在user_name 这一列进行groupby怎么办？ 如果确实需要这样做的话，那么需要对groupby的字段进行一个处理，以确保就是这个集合是可以在分组之后都是唯一的(可以理解为只有一行)

### 二：
除了上面的方法，还可以对分组查询出来的数据进行一个聚合操作。
```
mysql> SELECT user_name ,COUNT(*) AS 'subject_num' FROM testgroupby  GROUP BY  user_name;
+-----------+-------------+
| user_name | subject_num |
+-----------+-------------+
| 张三      |           3 |
| 李四      |           2 |
| 王五      |           2 |
+-----------+-------------+
3 rows in set (0.00 sec)

```
对于聚合之后的操作，MySQL是接受查询的
### 三：ANY_VALUE()函数：
```
mysql> select ANY_VALUE(user_id),user_name  from testgroupby group by user_name;
+--------------------+-----------+
| ANY_VALUE(user_id) | user_name |
+--------------------+-----------+
|                  1 | 张三      |
|                  4 | 李四      |
|                  5 | 王五      |
+--------------------+-----------+
3 rows in set (0.00 sec)

```
当然这样使用的话mysql只会取所分组得第一行。
### GROUP_CONCAT()函数：
这个函数会将一个查询得结果集进行合并，从而可以使对user_name进行groupby之后返回得是一行
```
mysql> SELECT GROUP_CONCAT(user_id) AS 'user_id',user_name FROM testgroupby GROUP BY user_name;
+---------+-----------+
| user_id | user_name |
+---------+-----------+
| 1,2,3   | 张三      |
| 4,6     | 李四      |
| 5,7     | 王五      |
+---------+-----------+
3 rows in set (0.00 sec)

```

## 总结：
### 关于这个模式：
下面这个图大致的解释了下为什么会报出这个错误。
![](mysql的疑惑.PNG)
因为最后mysql会疑惑，你分组之后那么多得数据，我知道选则分组之后得哪一行？？？
