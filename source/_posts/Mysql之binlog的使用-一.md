---
title: Mysql之binlog的使用(一)
date: 2018-11-11 22:37:55
tags: [mysql]
categories: mysql
---
## 背景

假设有一个如下的业务场景如下：
需要记录一个商品或者股票的实时价格，每一个小时记录一次，而商品或者股票的数量十分多，这时业务发展到一定的程度之后就需要考虑数据库的设计。首先商品每个小时的价格肯定是需要入库的。其次每小时的购买人群以及各种埋点数据随之一起也要入库。以便于日后的数据分析。

## 解决方案
随着数据量的增大，一般的解决方案是设置索引，然后再考虑是进行垂直还是水平分库分表。


但是一旦使用水平分库分表就会无形之间增加开发的复杂程度，而且分库分表之后考虑的各种因素也会随之而来增加数倍。例如各种表的`唯一ID`以及如何进行维度的划分。


对于数据量不大的一个另解决方法是：解析`mysql`的binlog日志，然后将其存入另一个库，优先推荐`mongo`，该库只作为一个读取库进行查询，不进行任何的写入。这样处理之后，在中等程度的规模数据是完全可以满足需求的，将其读写进行分离。而且丝毫不影响之前的业务和设计。


## binlog
其实对于增删改查之外，`mysql`的binlog日志也是一个非常有用的工具，它可以记录下数据库的每一次操作，例如查询，新增，删除，更新等。然后将其作为日志记录在binlog之中。

## 例子
首先确保你的数据库已经开启了`binlog`日志，
```sql
mysql> show variables like 'log_%';
+----------------------------------------+------------------------------------------------------------------+
| Variable_name                          | Value                                                            |
+----------------------------------------+------------------------------------------------------------------+
| log_bin                                | ON                                                               |
| log_bin_basename                       | /opt/mysql/mysql-8.0.11-linux-glibc2.12-x86_64/data/master       |
| log_bin_index                          | /opt/mysql/mysql-8.0.11-linux-glibc2.12-x86_64/data/master.index |
| log_bin_trust_function_creators        | OFF                                                              |
| log_bin_use_v1_row_events              | OFF                                                              |
| log_error                              | ./error.log                                                      |
| log_error_services                     | log_filter_internal; log_sink_internal                           |
| log_error_verbosity                    | 2                                                                |
| log_output                             | FILE                                                             |
| log_queries_not_using_indexes          | OFF                                                              |
| log_slave_updates                      | ON                                                               |
| log_slow_admin_statements              | OFF                                                              |
| log_slow_slave_statements              | OFF                                                              |
| log_statements_unsafe_for_binlog       | ON                                                               |
| log_syslog                             | ON                                                               |
| log_syslog_facility                    | daemon                                                           |
| log_syslog_include_pid                 | ON                                                               |
| log_syslog_tag                         |                                                                  |
| log_throttle_queries_not_using_indexes | 0                                                                |
| log_timestamps                         | UTC                                                              |
+----------------------------------------+------------------------------------------------------------------+
20 rows in set (0.01 sec)
```
可以看到`| log_bin                                | ON           `，这就表示数据库已经开启了`binlog`日志，如果你查询到还没有开启的话，可以去搜索下如何开启。

另外需要注意的是，如果你需要在`binlog`中看到日志的话，你同时也需要在Mysql中设置`binlog_rows_query_log_events`为`Row`，如果不确定的话，可以将`set binlog_rows_query_log_events=1`执行一次。


为了测试，已经建立好了一个表，其结构与如下：
```sql
mysql> desc T_fund;
+-------+---------------+------+-----+---------+-----------------------------+
| Field | Type          | Null | Key | Default | Extra                       |
+-------+---------------+------+-----+---------+-----------------------------+
| id    | int(10)       | NO   | PRI | NULL    |                             |
| name  | varchar(255)  | YES  |     | NULL    |                             |
| price | decimal(10,2) | YES  |     | NULL    |                             |
| date  | timestamp     | NO   |     | NULL    | on update CURRENT_TIMESTAMP |
+-------+---------------+------+-----+---------+-----------------------------+
4 rows in set (0.01 sec)

```


那么现在尝试向该表插入一个语句，
```sql
mysql>   insert into T_fund(id,name,price,date) values(3,'测试基金3',1234.1,'2018-11-11 22:12:00');
Query OK, 1 row affected (0.09 sec)

```
此时查看binlog，
```sql
mysql> show master logs;
+---------------+-----------+
| Log_name      | File_size |
+---------------+-----------+
| master.000001 |       178 |
| master.000002 |     62644 |
| master.000003 |    112942 |
+---------------+-----------+
3 rows in set (0.00 sec)

```

可以看到目前是有三个`binlog`文件，由于binlog是依次从1开始递增，所以刚刚的插入语句是在第三个日志中，查看`master.000003`即可。

```sql
mysql>  show binlog events in 'master.000003' from 114213\G;
*************************** 1. row ***************************
   Log_name: master.000003
        Pos: 114213
 Event_type: Rows_query
  Server_id: 1
End_log_pos: 114330
       Info: # insert into T_fund(id,name,price,date) values(3,'测试基金3',1234.1,'2018-11-11 22:12:00')
*************************** 2. row ***************************
   Log_name: master.000003
        Pos: 114330
 Event_type: Table_map
  Server_id: 1
End_log_pos: 114397
       Info: table_id: 179 (T_binlog.T_fund)
*************************** 3. row ***************************
   Log_name: master.000003
        Pos: 114397
 Event_type: Write_rows
  Server_id: 1
End_log_pos: 114461
       Info: table_id: 179 flags: STMT_END_F
*************************** 4. row ***************************
   Log_name: master.000003
        Pos: 114461
 Event_type: Xid
  Server_id: 1
End_log_pos: 114492
       Info: COMMIT /* xid=4004 */
4 rows in set (0.00 sec)

```
由于我知道该条日志的`position`，所以加了一个参数`from 114213`，如果不知道的话，可以直接输入`show binlog events in 'master.000003`然后看结尾即可。 

有了`mysql`的binlog之后，下一步就需要考虑如何将`binlog`解析到其他的数据库之中，目前开源的轮子，比较好的有:

> [Maxwell(github)](https://github.com/zendesk/maxwell)<br>
  [canal(alibaba)](https://github.com/alibaba/canal)
  
  
  在这里推荐Maxwell，因为Maxwell目前已经对`MySql8`有一个比较好的支持了，而且我们已经将部分业务应用到了Maxwell了，目前还比较稳定。
  
  下一篇主要会将`Maxwell`与`binlog`以及`rabbitmq`进行一个整合。