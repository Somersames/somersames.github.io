---
title: mysql的一些总结
date: 2018-03-29 22:03:32
tags: [MySql]
categories: [数据库,MySql]
---
## 前言
今天突然有一个写sql的机会，但是是手写，不像之前那样可以在数据库上做测试。这突然让我感觉有的语法有点生疏了，所以乘着这个机会来做一个全部的梳理。

## groupby和where的顺序：
今天是有两表做一个等值连接查询的，在这里应该是先where之后再进行group by，`group by `是对`where`条件过滤之后再进行分组处理，所以where在前。

### group by
在这里之前一直对group by有一个比较错误的认识，先看一下表结构：
```
mysql> SELECT e.employeeNumber,e.firstName ,e.extension, o.state ,o.officeCode FROM employees e ,offices  o WHERE e.officeCode = o.officeCode;
+----------------+-----------+-----------+------------+------------+
| employeeNumber | firstName | extension | state      | officeCode |
+----------------+-----------+-----------+------------+------------+
|           1002 | Diane     | x5800     | CA         | 1          |
|           1056 | Mary      | x4611     | CA         | 1          |
|           1076 | Jeff      | x9273     | CA         | 1          |
|           1143 | Anthony   | x5428     | CA         | 1          |
|           1165 | Leslie    | x3291     | CA         | 1          |
|           1166 | Leslie    | x4065     | CA         | 1          |
|           1188 | Julie     | x2173     | MA         | 2          |
|           1216 | Steve     | x4334     | MA         | 2          |
|           1286 | Foon Yue  | x2248     | NY         | 3          |
|           1323 | George    | x4102     | NY         | 3          |
|           1102 | Gerard    | x5408     | NULL       | 4          |
|           1337 | Loui      | x6493     | NULL       | 4          |
|           1370 | Gerard    | x2028     | NULL       | 4          |
|           1401 | Pamela    | x2759     | NULL       | 4          |
|           1702 | Martin    | x2312     | NULL       | 4          |
|           1621 | Mami      | x101      | Chiyoda-Ku | 5          |
|           1625 | Yoshimi   | x102      | Chiyoda-Ku | 5          |
|           1088 | William   | x4871     | NULL       | 6          |
|           1611 | Andy      | x101      | NULL       | 6          |
|           1612 | Peter     | x102      | NULL       | 6          |
|           1619 | Tom       | x103      | NULL       | 6          |
|           1501 | Larry     | x2311     | NULL       | 7          |
|           1504 | Barry     | x102      | NULL       | 7          |
+----------------+-----------+-----------+------------+------------+
23 rows in set (0.00 sec)
```
在这之前一直是想通过groupby达到如下的效果：
```
mysql> SELECT e.employeeNumber,e.firstName ,e.extension, o.state ,o.officeCode FROM employees e ,offices  o WHERE e.officeCode = o.officeCode and o.officeCode=3;
+----------------+-----------+-----------+-------+------------+
| employeeNumber | firstName | extension | state | officeCode |
+----------------+-----------+-----------+-------+------------+
|           1286 | Foon Yue  | x2248     | NY    | 3          |
|           1323 | George    | x4102     | NY    | 3          |
+----------------+-----------+-----------+-------+------------+
2 rows in set (0.00 sec)
```
其实这种只能是通过一个内连接加一个条件过滤来进行实现，但是在之前的话是一直在记忆里面认为groupby可行，其实**是不可行的** ，为什么这样说了，因为group by按照条件分组之后他每组只会有一条，那么在这里就出现了一个问题，既然groupby之后显示的是一条，那么这两行如何显示呢？所以这种通过groupby来显示的是不可行的。
那么，groupby适合什么呢？groupby适合的是按照分组进行统计的，例如求最大薪水，平均薪水等，而不是列出某一个分组里面的所有人的。列出所有人这是where擅长的，如下数据表：
```
mysql> SELECT o.orderNUmber, o.orderDate, od.priceEach FROM orders o ,orderdetails od WHERE o.orderNumber =od.orderNumber limit 0,10;
+-------------+------------+-----------+
| orderNUmber | orderDate  | priceEach |
+-------------+------------+-----------+
|       10100 | 2003-01-06 |    136.00 |
|       10100 | 2003-01-06 |     55.09 |
|       10100 | 2003-01-06 |     75.46 |
|       10100 | 2003-01-06 |     35.29 |
|       10101 | 2003-01-09 |    108.06 |
|       10101 | 2003-01-09 |    167.06 |
|       10101 | 2003-01-09 |     32.53 |
|       10101 | 2003-01-09 |     44.35 |
|       10102 | 2003-01-10 |     95.55 |
|       10102 | 2003-01-10 |     43.13 |
+-------------+------------+-----------+
10 rows in set (0.00 sec)

```
那么按照orderNumber 进行groupby便可以求出每一个订单的总价了，也说明了groupby只是适合于分组统计，而不是适用于分组展示：
```
SELECT o.orderNUmber, o.orderDate ,SUM(od.priceEach) FROM orders o ,orderdetails od WHERE o.orderNumber =od.orderNumber GROUP BY od.orderNumber limit 0,10;
+-------------+------------+-------------------+
| orderNUmber | orderDate  | SUM(od.priceEach) |
+-------------+------------+-------------------+
|       10100 | 2003-01-06 |            301.84 |
|       10101 | 2003-01-09 |            352.00 |
|       10102 | 2003-01-10 |            138.68 |
|       10103 | 2003-01-29 |           1520.37 |
|       10104 | 2003-01-31 |           1251.89 |
|       10105 | 2003-02-11 |           1479.71 |
|       10106 | 2003-02-17 |           1427.28 |
|       10107 | 2003-02-24 |            793.21 |
|       10108 | 2003-03-03 |           1432.86 |
|       10109 | 2003-03-10 |            700.89 |
+-------------+------------+-------------------+
10 rows in set (0.01 sec)
```
这才是groupby的正确用法

### having count函数：
`having count`函数可以配合group by 对分组之后的数据进行筛选，这是where过滤所达不到的，就比如上表，需要过滤出订单价格大于1000的，那么在用where的时候则会显得很乏力：
```
mysql> SELECT o.orderNUmber, o.orderDate ,SUM(od.priceEach) AS total FROM orders o ,orderdetails od WHERE o.orderNumber =od.orderNumber GROUP BY od.orderNumber HAVING TOTAL > 1000 LIMIT 0,10;
+-------------+------------+---------+
| orderNUmber | orderDate  | total   |
+-------------+------------+---------+
|       10103 | 2003-01-29 | 1520.37 |
|       10104 | 2003-01-31 | 1251.89 |
|       10105 | 2003-02-11 | 1479.71 |
|       10106 | 2003-02-17 | 1427.28 |
|       10108 | 2003-03-03 | 1432.86 |
|       10110 | 2003-03-18 | 1338.47 |
|       10117 | 2003-04-16 | 1307.47 |
|       10119 | 2003-04-28 | 1081.44 |
|       10120 | 2003-04-29 | 1322.67 |
|       10122 | 2003-05-08 | 1598.27 |
+-------------+------------+---------+
10 rows in set (0.00 sec)
```
这就是having配合groupby的便捷性，那么where适合什么形式的过滤呢？我个人认为where比较适合于在groupby之前就进行数据的过滤，这样会比较好，而having count则适合于groupby之后的一个过滤。

### OrderBy函数：
orderby函数可以对结果集进行排序，他既可以和where搭配使用，又可以和groupby搭配使用，又可以和having一起搭配使用：
```
mysql> SELECT o.orderNUmber, o.orderDate ,SUM(od.priceEach) AS total FROM orders o ,orderdetails od WHERE o.orderNumber =od.orderNumber GROUP BY od.orderNumber HAVING TOTAL > 1000 order by total  LIMIT 0,10;
+-------------+------------+---------+
| orderNUmber | orderDate  | total   |
+-------------+------------+---------+
|       10341 | 2004-11-24 | 1003.19 |
|       10293 | 2004-09-09 | 1004.59 |
|       10246 | 2004-05-05 | 1006.78 |
|       10420 | 2005-05-29 | 1014.01 |
|       10311 | 2004-10-16 | 1033.82 |
|       10380 | 2005-02-16 | 1034.10 |
|       10412 | 2005-05-03 | 1034.15 |
|       10278 | 2004-08-06 | 1034.86 |
|       10361 | 2004-12-17 | 1052.87 |
|       10271 | 2004-07-20 | 1054.03 |
+-------------+------------+---------+
10 rows in set (0.01 sec)

```
## Mysql的总结：

其实mysql的连接无非是左连接，右连接，内连接，全连接，也就left join ,right join ,inner join 和union等，
其实在这里group by 和having count都可以对多列进行一个排序或者过滤，而不仅仅是一行