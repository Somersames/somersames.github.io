---
title: Mysql中的幻读(一)
date: 2019-08-18 20:41:54
tags: [Mysql]
categories: Mysql
---
## 什么是幻读

幻读表示的是在一个事物里面 同一个`select`语句，前后两次查询出来的结果是不相同的，需要注意的一点是，在InnoDB里面，幻读跟事物的隔离级别有关，更加准确的说是跟一个事物的快照和当前读有关

下面是在Mysql8.0.11版本下进行幻读的复现：

- 引擎：InnoDB
- 事物隔离级别：Read Commited



### MVCC和快照读以及当前读

#### MVCC


在介绍MVCC之前先来介绍下MVCC为什么会出现，首先数据库作为一个数据存储工具，那么肯定是存在并发的情况，
在Mysql的InnoDB里面最常见的就是`x`锁，这是一种写锁，在并发的情况下只有一个事务会获取该锁，其他事务则会一直等待直至获取到该锁。

那么在读取的时候如何保证并发的事物都能正确的读取到自己正确的数据呢？

在MVCC的概念里面，如果事务的隔离级别是`Read Commited`的话，那么每一次的快照都都会读取该行的最近一次`commited`数据，而如果是`Repeatable Read`的话，则是会读取当前事务ID开始之前的一次`commited`数据。

所以MVCC仅仅是作为一个保证数据库并发读情况下的一个数据正确的手段而已，在不同的数据库里面，有不同的实现，例如在`OceanBase`里面，是通过操作链来解决并发读的问题
#### 快照读
快照读是利用MVCC和 `undo log` 来实现的，其主要作用就是当我们对某行数据修改之后，并不会将原值修改，而是在上一个版本上面再新建一个版本(修改InnoDB的隐藏两列)。
所以在不同的隔离级别下，可以根据自己的`事物ID`来获取自己所需要的数据。当第一条不满足的时候，会沿着`undo log`一直寻找，在`Read Commited`隔离级别下就是直接找出该行数据最后一次提交的版本

#### 当前读
当前读是对数据的加锁读取，读取的都是最新的数据，例如如下SQL：
```mysql
select ... for update
select ... in share mode
insert
update
delete
...
```
同时也需要注意的一点是，当前读会对涉及到的行都进行加锁
#### 为什么会有当前读和快找读
按照Mysql官方的解释，当前读是为了防止其他事物修改你即将进行的操作

> If you query data and then insert or update related data within the same transaction, the regular SELECT statement does not give enough protection. Other transactions can update or delete the same rows you just queried. InnoDB supports two types of locking reads that offer extra safety

### 幻读发生的条件
首先解释下Mysql官方对于幻读的解释：
> The so-called phantom problem occurs within a transaction when the same query produces different sets of rows at different times. For example, if a SELECT is executed twice, but returns a row the second time that was not returned the first time, the row is a “phantom” row.

这里Mysql官方虽然只是给出了对于`幻行`的定义，但是仍然可以简单解释下，也就是说两次的`select`前后得到的结果集是不同的，那么新多出来的一行就可以称之为`幻行`

**在这里特别需要注意的是，在PR隔离界别下，只有当前读才会出现幻读**

#### Read Commited
在`Read Commited`隔离级别下，每一次的`select` 都是一次快照读，所以在该隔离级别下，幻读可以在快照读和当前读发生。
> 事务一
```MYSQL
mysql> set session transaction isolation level read committed;
Query OK, 0 rows affected (0.00 sec)

mysql> START TRANSACTION;
Query OK, 0 rows affected (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from szh.t1
    -> ;
+---------+-----------+
| area_id | order_num |
+---------+-----------+
|       1 |        22 |
|       2 |        10 |
+---------+-----------+
2 rows in set (0.00 sec)

mysql> select * from `szh`.`t1`;
+---------+-----------+
| area_id | order_num |
+---------+-----------+
|       1 |        22 |
|       2 |        10 |
+---------+-----------+
2 rows in set (0.00 sec)

```
> 事务二
```mysql
INSERT INTO `szh`.`t1`(`area_id`, `order_num`) VALUES (3, 22);
```
此时`事务一`再进行select
```mysql
mysql> select * from `szh`.`t1`;
+---------+-----------+
| area_id | order_num |
+---------+-----------+
|       1 |        22 |
|       2 |        10 |
|       3 |        22 |
+---------+-----------+
3 rows in set (0.00 sec)

```
于是幻读发生了

#### Repeatable Read
那么如果在`Repeatable Read`隔离级别下，上面的SQL在执行一遍会出现什么情况呢？
> 事务一
```mysql
mysql> START TRANSACTION;
Query OK, 0 rows affected (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT * FROM `t1`
    -> ;
+---------+-----------+
| area_id | order_num |
+---------+-----------+
|       1 |        22 |
|       2 |        10 |
|       3 |        22 |
|       4 |        33 |
|       5 |        55 |
+---------+-----------+
5 rows in set (0.00 sec)

```
> 事务二
```mysql
INSERT INTO `szh`.`t1`(`area_id`, `order_num`) VALUES (6, 66);
```
> 事务一再次select
```mysql
mysql> SELECT * FROM `t1`;
+---------+-----------+
| area_id | order_num |
+---------+-----------+
|       1 |        22 |
|       2 |        10 |
|       3 |        22 |
|       4 |        33 |
|       5 |        55 |
+---------+-----------+
5 rows in set (0.00 sec)

```

可以看到的是在快照读下面，是没有幻读出现的，那么修改select为当前读呢？
在事务一里面再次执行以下SQL
```mysql
mysql> SELECT * FROM `t1` for update;
+---------+-----------+
| area_id | order_num |
+---------+-----------+
|       1 |        22 |
|       2 |        10 |
|       3 |        22 |
|       4 |        33 |
|       5 |        55 |
|       6 |        66 |
+---------+-----------+
6 rows in set (0.00 sec)
```
可以看到确实出现了两次的select不同的情况。
所以需要强调一点的是，在`Repeatable Read`隔离级别下，只有当前读才会出现幻读，因为在该级别下，快照读是从`begin`开始的第一个普通`select`建立的`Read View`，以后的普通`select`都是基于第一次的`select`，自然而然不会出现幻读了 