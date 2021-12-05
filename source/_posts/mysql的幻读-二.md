---
title: MySql的幻读(二)
date: 2019-08-27 23:48:53
tags: [MySql]
categories: [数据库,MySql]
---

## 定义
在InnoDB里面，是通过快照读来实现`RC`和`PR`隔离级别的区分，因为在`RC`隔离级别下，每一次的select都是一个快照读，所以是可以读取到已经提交的数据，从而导致幻读。所以在`RC`隔离级别下，快照读和当前读都是可以出现幻读。

但是在`PR`的隔离级别下，由于快照读仅仅只生成一次，所以在`PR`级别下的快照读是无法出现幻读的，但是当前读确实可以出现幻读。

查看Mysql官方对于`幻读`的定义。
> The so-called phantom problem occurs within a transaction when the same query produces different sets of rows at different times. 

也就是说一个事物同一条查询语句查询出来了两个不同的集合就可以称之为幻读。


### ANSI SQL 隔离级别
在`ANSI SQL 隔离级别`的定义中`PR`级别是可以出现幻读的。
但是在InnoDB引擎里面自己通过`GAP锁`和`Next-Key锁`使得`PR`隔离级别下无法出现幻读。原因就在于InnoDB的`GAP`锁


## GAP锁
在`InnoDB`里面，GAP锁是用于防止幻读的一个手段，具体的操作是首先当我们新增一个数据的时候，`InnoDB`会在此时加一个`GAP`锁从而防止其他事务对该区间的一些数据操作，导致幻读的出现。

### 复现
#### `PR`级别：
```mysql
mysql> select * from test_lock.test;
+---------+-----------+----------------+------------+-------------+
| user_id | user_name | user_password  | is_deleted | phone       |
+---------+-----------+----------------+------------+-------------+
| a       | zhangsan  | 123456         |          0 | 15112345678 |
| b       | lisi      | lisi123456     |          0 | 15112345678 |
| c       | wangwu    | wangwu123456   |          0 | 15112345678 |
| d       | caocao    | caocao123456   |          0 | 15112345678 |
| e       | liubei    | liubei123456   |          0 | 15112345678 |
| f       | zhangfei  | zhangfei123456 |          0 | 15112345678 |
| g       | guanyu    | guanyu123456   |          0 | 15112345678 |
| h       | daqiao    | daqiao123456   |          0 | 15112345678 |
| m       | xiaoqiao  | xiaoqiao123456 |          0 | 15112345678 |
+---------+-----------+----------------+------------+-------------+
9 rows in set (0.00 sec)
```

*事物一*：
```mysql
mysql> start transaction ;
Query OK, 0 rows affected (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from test_lock.test where  user_id > 'g' and user_id < 'm' for update ;
+---------+-----------+---------------+------------+-------------+
| user_id | user_name | user_password | is_deleted | phone       |
+---------+-----------+---------------+------------+-------------+
| h       | daqiao    | daqiao123456  |          0 | 15112345678 |
+---------+-----------+---------------+------------+-------------+
1 row in set (0.00 sec)
```
*事物二*：
```mysql
mysql> start transaction ;
Query OK, 0 rows affected (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into test_lock.`test` values('i','daqiao','daqiao123456',0,'15112345678');
```
此时在`PR`隔离级别下会阻塞。


此时查看数据库中的锁：
```mysql
ysql> select * from performance_schema.data_locks;
+--------+----------------------------------------+-----------------------+-----------+----------+---------------+-------------+----------------+-------------------+------------+-----------------------+-----------+------------------------+-------------+-----------+
| ENGINE | ENGINE_LOCK_ID                         | ENGINE_TRANSACTION_ID | THREAD_ID | EVENT_ID | OBJECT_SCHEMA | OBJECT_NAME | PARTITION_NAME | SUBPARTITION_NAME | INDEX_NAME | OBJECT_INSTANCE_BEGIN | LOCK_TYPE | LOCK_MODE              | LOCK_STATUS | LOCK_DATA |
+--------+----------------------------------------+-----------------------+-----------+----------+---------------+-------------+----------------+-------------------+------------+-----------------------+-----------+------------------------+-------------+-----------+
| INNODB | 140169161014704:1068:140169076344152   |                235275 |        50 |       16 | test_lock     | test        | NULL           | NULL              | NULL       |       140169076344152 | TABLE     | IX                     | GRANTED     | NULL      |
| INNODB | 140169161014704:7:4:14:140169076341272 |                235275 |        50 |       16 | test_lock     | test        | NULL           | NULL              | PRIMARY    |       140169076341272 | RECORD    | X,GAP,INSERT_INTENTION | WAITING     | 'm'       |
| INNODB | 140169161013840:1068:140169076338200   |                235274 |        47 |       20 | test_lock     | test        | NULL           | NULL              | NULL       |       140169076338200 | TABLE     | IX                     | GRANTED     | NULL      |
| INNODB | 140169161013840:7:4:14:140169076335256 |                235274 |        47 |       20 | test_lock     | test        | NULL           | NULL              | PRIMARY    |       140169076335256 | RECORD    | X                      | GRANTED     | 'm'       |
| INNODB | 140169161013840:7:4:15:140169076335256 |                235274 |        47 |       20 | test_lock     | test        | NULL           | NULL              | PRIMARY    |       140169076335256 | RECORD    | X                      | GRANTED     | 'h'       |
+--------+----------------------------------------+-----------------------+-----------+----------+---------------+-------------+----------------+-------------------+------------+-----------------------+-----------+------------------------+-------------+-----------+
5 rows in set (0.00 sec)

```
可以看到在`m`列由于`GAP`锁和`INSERT_INTENTION`互相冲突，导致`事物二`无法进行插入
> 事物一由于进行`for update`查询，所以会对区间加一个GAP锁
> 事物二由于是新增，所以会加一个插入意向锁
> 由于`GAP`锁阻塞住了插入意向锁，导致`事物二`无法进行插入

此时`事物二`也会在这里阻塞住，而在`RC`隔离级别下，`事物二`是不会等待的。
但是如果仔细一点，其实这里还可以发现另一个锁，就是插入意向锁`INSERT_INTENTION LOCK`

## INSERT_INTENTION_LOCK
Mysql官方文档中将`INSERT_INTENTION`定义为一个`GAP`锁，但是它的意义和真正的`GAP`锁之间是有天大的差别的，`INSERT_INTENTION_LOCK`并不会阻塞`GAP`锁，但相反`GAP`会阻塞`INSERT_INTENTION_LOCK`，并且该锁的锁定范围是插入行一直到下一个索引，这一整个区间
如下例子：
*事物一*：
```mysql
mysql> start transaction ;
Query OK, 0 rows affected (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into test_lock.`test` values('i','daqiao','daqiao123456',0,'15112345678');
Query OK, 1 row affected (0.00 sec)
```
*事物二*：
```mysql
mysql> start transaction ;
Query OK, 0 rows affected (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> update test_lock.`test` set user_id='l'  where  user_id = 'k';
Query OK, 0 rows affected (0.00 sec)
Rows matched: 0  Changed: 0  Warnings: 0
```

此时查看事物中锁：
```mysql
mysql> select * from performance_schema.data_locks;
+--------+----------------------------------------+-----------------------+-----------+----------+---------------+-------------+----------------+-------------------+------------+-----------------------+-----------+-----------+-------------+-----------+
| ENGINE | ENGINE_LOCK_ID                         | ENGINE_TRANSACTION_ID | THREAD_ID | EVENT_ID | OBJECT_SCHEMA | OBJECT_NAME | PARTITION_NAME | SUBPARTITION_NAME | INDEX_NAME | OBJECT_INSTANCE_BEGIN | LOCK_TYPE | LOCK_MODE | LOCK_STATUS | LOCK_DATA |
+--------+----------------------------------------+-----------------------+-----------+----------+---------------+-------------+----------------+-------------------+------------+-----------------------+-----------+-----------+-------------+-----------+
| INNODB | 140169161014704:1068:140169076344152   |                235293 |        50 |       48 | test_lock     | test        | NULL           | NULL              | NULL       |       140169076344152 | TABLE     | IX        | GRANTED     | NULL      |
| INNODB | 140169161014704:7:4:14:140169076341272 |                235293 |        50 |       48 | test_lock     | test        | NULL           | NULL              | PRIMARY    |       140169076341272 | RECORD    | X,GAP     | GRANTED     | 'm'       |
| INNODB | 140169161013840:1068:140169076338200   |                235288 |        47 |       32 | test_lock     | test        | NULL           | NULL              | NULL       |       140169076338200 | TABLE     | IX        | GRANTED     | NULL      |
+--------+----------------------------------------+-----------------------+-----------+----------+---------------+-------------+----------------+-------------------+------------+-----------------------+-----------+-----------+-------------+-----------+
3 rows in set (0.00 sec)
```
可以看到`事物二`确实已经执行成功了，而且`事物一`的插入意向锁并未阻塞事物二的插入语句
。

## 总结
在Mysql的`InnoDB`中，幻读的解决方案是采用了一个间隙锁`GAP`锁来实现的