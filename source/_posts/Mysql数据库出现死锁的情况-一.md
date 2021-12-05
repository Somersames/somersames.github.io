---
title: MySql数据库出现死锁的情况(一)
date: 2019-08-04 16:32:31
tags: [Mysql]
categories: [数据库,MySql]
---
在临近上线之前，我们系统做了一次压力测试，发现有一个接口在高并发情况下会出现一个死锁的情况。。首先申明...不是我写的，我只是帮忙排查下。

随着对Mysql锁的深入了解，于是就准备写几篇文章来记录下Mysql各种事物和索引的情况下出现死锁的情况。



今天就介绍下在并发插入的情况下，哪几种情况会出现死锁：

## INNODB下的各种锁

在介绍锁的时候只会介绍跟本节相关的锁，而且只会讲述大概是什么，至于锁的更加详细的讲解可能会到以后再详细介绍。

### 行锁

行锁分为写锁和读取锁，

> 读锁（S锁）也可以称之为 `共享锁` ， 它表示的是任何一个事物都可以读取该行数据(可以被多个事物获取到)。

> 写锁（X锁）也可以称之为排它锁，它表示的是该行数据不允许任何人进行修改，同时也不允许任何事物获取该行事物的S锁，但是普通的 `select` 语句是可以的。

## 背景信息一

### 注意：以下测试都是基于该事物隔离级别和数据库版本

数据库版本：8.0.11

事务隔离级别：REPEATABLE-READ

```sql
mysql> SHOW VARIABLES LIKE 'transaction_isolation';
+-----------------------+-----------------+
| Variable_name         | Value           |
+-----------------------+-----------------+
| transaction_isolation | REPEATABLE-READ |
+-----------------------+-----------------+
1 row in set (0.00 sec)
```

## SQL准备:

```sql
create database if not exists test_lock  DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
CREATE TABLE test_lock.`test` (
    `user_id` varchar(32) NOT NULL,
    `user_name` varchar(32) NOT NULL,
    `user_password` varchar(16) NOT NULL,
    `is_deleted` int(1) NOT NULL,
    `phone` varchar(20) NOT NULL,
    PRIMARY KEY (`user_id`),
    KEY `t1_god` (`user_name`,`user_password`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

准备数据：

```sql
insert into test_lock.`test` values('a','zhangsan','123456',0,'15112345678');
insert into test_lock.`test` values('b','lisi','lisi123456',0,'15112345678');
insert into test_lock.`test` values('c','wangwu','wangwu123456',0,'15112345678');
insert into test_lock.`test` values('d','caocao','caocao123456',0,'15112345678');
insert into test_lock.`test` values('e','liubei','liubei123456',0,'15112345678');
insert into test_lock.`test` values('f','zhangfei','zhangfei123456',0,'15112345678');
insert into test_lock.`test` values('g','guanyu','guanyu123456',0,'15112345678');
insert into test_lock.`test` values('h','xiaoqiao','xiaoqiao123456',0,'15112345678');
```

### 情况一：

>  三个事物都执行同一个Insert语句，第一个事物然后回滚

此时第三个事物会提示死锁，第二个事物正常插入。

#### 复现步骤：

开启三个事物，每一个事物分别执行如下SQL，然后再将第一个事物进行回滚或者提交，然后你就会发现第三个事物发生了死锁。

```sql
## DeadLock
start transaction ;
begin;
insert into test_lock.`test` values('i','daqiao','daqiao123456',0,'15112345678');
```

Mysql提示如下：

> [2019-08-04 14:33:48] Connected
> sql> start transaction
> [2019-08-04 14:33:49] completed in 4 ms
> sql> begin
> [2019-08-04 14:33:49] completed in 6 ms
> sql> insert into test_lock.`test` values('i','daqiao','daqiao123456',0,'15112345678')
> [2019-08-04 14:34:00] [40001][1213] Deadlock found when trying to get lock; try restarting transaction
> [2019-08-04 14:34:00] [40001][1213] Deadlock found when trying to get lock; try restarting transaction



那么打印出Mysql出现死锁的日志：重点日志如下

```verilog
LATEST DETECTED DEADLOCK
------------------------
2019-08-04 06:33:59 0x7f1818599700
*** (1) TRANSACTION:
TRANSACTION 231690, ACTIVE 34 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 4 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 9, OS thread handle 139741464762112, query id 95 172.17.0.1 root update
/* ApplicationName=IntelliJ IDEA 2019.1.3 */ insert into test_lock.`test` values('i','daqiao','daqiao123456',0,'15112345678')
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 7 page no 4 n bits 80 index PRIMARY of table `test_lock`.`test` trx id 231690 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

*** (2) TRANSACTION:
TRANSACTION 231691, ACTIVE 10 sec inserting
mysql tables in use 1, locked 1
4 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 10, OS thread handle 139741464467200, query id 139 172.17.0.1 root update
/* ApplicationName=IntelliJ IDEA 2019.1.3 */ insert into test_lock.`test` values('i','daqiao','daqiao123456',0,'15112345678')
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 7 page no 4 n bits 80 index PRIMARY of table `test_lock`.`test` trx id 231691 lock mode S
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 7 page no 4 n bits 80 index PRIMARY of table `test_lock`.`test` trx id 231691 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

*** WE ROLL BACK TRANSACTION (2)
------------
```

从以上日志我们可以看到，事物一正在请求该行记录的 `X锁` ，事物二持有该行的`S锁`，但是也在等待获取该行的`X锁`。

关于Mysql的 `insert` 逻辑，可以大致理解为如果一个事物正在对一个记录进行 `insert`，此时 InnoDB 并不会主动的将其加一个锁，而是在主键索引上加一个 `trx_id`，

当第二个事物检测到该行记录正在被一个活跃的事物持有的时候，此时第二个事物会帮第一个事物的隐式锁住转为显式锁。如下例子：

```sql
## DeadLock
start transaction ;
begin;
insert into test_lock.`test` values('i','daqiao','daqiao123456',0,'15112345678');
```

此时查看Mysql中锁的情况：

![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/insert%E6%AD%BB%E9%94%81%E7%AC%AC%E4%B8%80%E4%B8%AA%E4%BA%8B%E7%89%A9.png)

可以看到此时Mysql仅仅是在表上加入了一个插入意向锁（IX锁），持有该锁表示该事物在接下来有可能会对自己设计到的行加入排它锁（X锁）

[Mysql关于意向锁的介绍](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)

> The intention locking protocol is as follows:
>
> - Before a transaction can acquire a shared lock on a row in a table, it must first acquire an `IS` lock or stronger on the table.
> - Before a transaction can acquire an exclusive lock on a row in a table, it must first acquire an `IX` lock on the table.

大意就是如果你想获得一个行的 `X锁`，那么你就必须先获取表的 `IX锁`

那么此时再进行第二个事物的插入：

```sql
## DeadLock
start transaction ;
begin;
insert into test_lock.`test` values('i','daqiao','daqiao123456',0,'15112345678');
```

再次查看数据库中的锁：

![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/Mysql%E7%AC%AC%E4%BA%8C%E4%B8%AA%E4%BA%8B%E7%89%A9.png)

此时会发现第一个事物已经获取到了行级别的`X锁`，第二个事物获取到了 `IX` 锁以及 `S锁` 。

那么此时第三个事物开启之后数据库中的锁又会是什么样子的呢？

![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/Mysql%E7%AC%AC%E4%B8%89%E4%B8%AA%E4%BA%8B%E7%89%A9.png)

可以发现另外两个事物都在等待获取该行的`S锁`。然后此时第一个事物进行回滚，此时第三个事物就会提示死锁。

## 原因

#### 前提结论

1. `S锁`是可以升级到`X锁`的
2. 一个`S锁`需要升级到`X`锁必须保证只有当前事物持有`S锁`

#### 死锁原因

当第一个事物回滚之后，第二个事物和第三个事物都会获得到该行的`S锁`，但是此时第二个事物将要执行 insert 语句，也就是第二个事物正在等待获取该行的`X锁`，但是此时第三个事物也正在准备 insert，它也在准备获取 `X锁`，目前这两个事物都持有该行的 `S锁` ，而如果需要获取 `X锁`，则是需要对方释放`S`锁。如下图：

![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/TwoTrxLock.png)

### 简单复现步骤：

为了验证上面的情况，我准备如下几个SQL：

```sql
## 事物一
start transaction ;
begin;
select * from test_lock.test where user_id='h'  lock in share mode;



## 事物二
start transaction ;
begin;
select * from test_lock.test where user_id='h'  lock in share mode;

## 事物一再执行
update test_lock.`test` set user_name='abc'  where  user_id = 'h';

## 事物二再执行
update test_lock.`test` set user_name='abc'  where  user_id = 'h';

## 此时事物二就会提示死锁
```

这是因为事物一和事物二都在等待对方释放`S锁`，但是都不肯释放，于是死锁发生了。

### 情况二

>  三个事物都执行同一个Insert语句，第一个事物然后提交

此时第二个事物和第三个事物都会提示主键冲突，并无死锁出现。



## 总结

这种情况下的死锁是由于 `S锁` 升级到 `X锁`导致的一种死锁，在平常的业务代码中应该尽量避免并发插入一个主键。