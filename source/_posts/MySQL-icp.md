---
title: MySQL_icp
toc: true
date: 2022-06-19 19:51:55
tags: [数据库]
categories: 数据库
---
如果没有明确的说明，本文的存储引擎均是 InnoDB，版本：8.0

假设一个表如下：

```sql
CREATE TABLE `dy_video_list` (
    `id` bigint unsigned NOT NULL AUTO_INCREMENT,
    `aweme_id` varchar(100) NOT NULL DEFAULT '',
    `collect_count` int NOT NULL DEFAULT '0',
    `comment_count` int NOT NULL DEFAULT '0',
    `digg_count` int NOT NULL DEFAULT '0',
    `share_count` int NOT NULL DEFAULT '0',
    `tag` varchar(100) NOT NULL DEFAULT '',
    `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    KEY `aweme_id_time` (`aweme_id`,`create_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci

```

目前数据有 2700 +

# 索引下推（ICP）

这是MySQL 在 5.6 版本所推出的一个功能，目的是为了减少回表次数，在 InnoDB 中，数据都是存储在 BufferPool 中，因此 InnoDB 中的索引下推不能减少 IO 次数，并且 InnoDB 中的索引下推仅仅适用于二级索引，无法用于主键索引。

在 MySQL 5.6 之前，如果 where 条件中，aweme_id 采用非等值查询，那么就会造成整个索引的失效。

例如我们先将 ICP 关掉

```sql
mysql> set optimizer_switch='index_condition_pushdown=off';
Query OK, 0 rows affected (0.00 sec)
```

然后执行一个查询：

![未开启ICP](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202206190021230.png)

然后此时我们再将 ICP 功能打开：

![开启ICP](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202206190021673.png)

## 工作原理

按照 MySQL 官方自己的解释是：

> Using Index Condition Pushdown, the scan proceeds like this instead:
>
> 1. Get the next row’s index tuple (but not the full table row).
> 2. Test the part of the `WHERE` condition that applies to this table and can be checked using only index columns. If the condition is not satisfied, proceed to the index tuple for the next row.
> 3. If the condition is satisfied, use the index tuple to locate and read the full table row.
> 4. Test the remaining part of the `WHERE` condition that applies to this table. Accept or reject the row based on the test result.
>    [`EXPLAIN`](https://dev.mysql.com/doc/refman/5.6/en/explain.html) output shows `Using index condition` in the `Extra` column when Index Condition Pushdown is used. It does not show `Using index` because that does not apply when full table rows must be read.

当通过「联合索引」进行查询的时候，如果未开启索引下推功能，存储引擎返回的数据都是需要服务端进行过滤，例如这个 SQL：

```sql
select * from dy_video_list where aweme_id like '61%' and create_time = 1655540046;

```

在未开启索引下推的功能时，会看到后面的 Extra 是 `Using where`，此时 InnoDB 会将所有 `61` 开头的数据通过回表查询，但同时也将 `create_time` 不是 1655540046 的数据也一起回表，并返回给服务端，服务端则根据 `where` 条件自己过滤。
![mariadb 介绍索引下推的图](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202206181635249.png)]

而开启索引下推以后，服务端将 where 过滤的功能下推至存储引擎，此时由 InnoDB 自己完成 where 过滤

![mariadb 介绍索引下推的图](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202206181638842.png)

# 意外情况

是不是觉得以后是一个索引都会走索引下推呢？那么如果把 SQL 改一下呢？

![全表扫描](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202206160020760.png)

此时是不是会发现竟然直接走了「全表扫描」？？说好的索引下推呢？

### 一探究竟

既然 MySQL 选择了全表扫描，那么我们不妨看下 MySQL 是如何来计算成本的，其中 MySQL 提供了 `OPTIMIZER_TRACE` 这个关键字来供大家分析，默认是 OFF 状态。

使用的时候，首先需要打开这个开关：

```sql
mysql> SET OPTIMIZER_TRACE="enabled=on";
Query OK, 0 rows affected (0.00 sec)
```

然后执行我们的SQL：

```sql
mysql> select * from dy_video_list where aweme_id like '6%' and create_time = 1655307436;

```

最后在执行：

```sql
select * from information_schema.OPTIMIZER_TRACE\G

```

此时就可以看到 MySQL 的控制台打印一大堆信息了。

![成本图](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202206160831709.png)

其中主要关心的是这三个节点的信息，其中第一个也就是 MySQL 认为全表扫描的成本，cost 为 266.8。

现在暂且不纠结该成本是怎么算出来的，先看下走 `aweme_id_name` 这个索引它的成本是多少，继续往下翻就可以找到。

![image-20220616083427243](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202206160834636.png)

可以看到成本是 297.06，于是 MySQL 认为直接走全表扫描的代码是小于走这个二级索引的，因此才会放弃。

那么为什么在这个情况下，MySQL 会选择放弃呢？，上图的截图中看到 `6%` 查询到的数据行数是 848 行，而这个表的总行数是多少呢？

![image-20220616084437866](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202206160844003.png)

大家可能发现这个数据和上面通过 `OPTIMIZER_TRACE` 分析的不一致，这个是由于 InnoDB 的特性，会有一定的差异，现在可以看到 848 大约是占比总行数的 30.6%。

那我给他改几条数据，让他小于 30% 看看

![image-20220616084952501](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202206160849902.png)

可以看到已经是小于 30% 了，此时再次分析看看

![image-20220616085206912](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202206190046188.png)

此时可以发现该查询已经可以通过索引下推来优化了

## 30%阈值

具体这个 30% 的阈值是怎么来的，目前是参考 oreilly 的一篇文章，具体的链接如下：
![High Performance MySQL by Jeremy D. Zawodny, Derek J. Balling](https://learning.oreilly.com/library/view/high-performance-mysql/0596003064/)

![image-20220616090238985](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202206160902651.png)

# 最后

如果有人问你索引下推一定会生效吗？可以直接跟他说，还得看索引的分散情况，如果联合索引的建立的不太好，那么索引下推也不会生效。
