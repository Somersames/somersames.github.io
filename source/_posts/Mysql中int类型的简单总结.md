---
title: MySql中int类型的简单总结
date: 2019-12-17 00:33:00
tags: [MySql]
categories: [数据库,MySql]
---
## 问题
首先问两个问题：
1. int(1)和int(10)有什么区别。
2. int(3)可以存储 10000 这个数字吗？
3. int(11)可以用来存储手机号么？

本次的源代码以及测试的 Mysql 版本均为 8.0.17

## 解释

首先新建一个表，SQL如下：
```sql
create table test_int(
    id int(3) not null primary key ,
    no int(5) not null,
    phone int(11)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```
当你执行完这个 SQL 以后，Mysql会发出一个如下提示：
> [2019-12-16 23:08:24] [HY000][1681] Integer display width is deprecated and will be removed in a future release.
首先不管这个提示，待会后文会解释的。


 然后新增测试数据：
 ```sql
 INSERT INTO `test`.`test_int` (`id`, `no`, `phone`) VALUES (10000, 1008611, 123124);
 INSERT INTO `test`.`test_int` (`id`, `no`, `phone`) VALUES (10, 134, 124);
 ```
 然后再进行 select 查看。
 ```sql
 mysql> select * from test_int;
+-------+---------+--------+
| id    | no      | phone  |
+-------+---------+--------+
|    10 |     134 |    124 |
| 10000 | 1008611 | 123124 |
+-------+---------+--------+
2 rows in set (0.00 sec)
 ```
 可以看到其实并没有影响到任何数据的插入，随后查询 Mysql 的官方文档，里面有这样的一段话
 > If you specify ZEROFILL for a numeric column, MySQL automatically adds the UNSIGNED attribute to the column.

随后再次新建一个表：
```sql
create table test_int_zerofill(
    id int(3) unsigned zerofill not null primary key ,
    no int(5) unsigned zerofill not null ,
    phone int(11) unsigned zerofill
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

The ZEROFILL attribute is deprecated and will be removed in a future release. Use the LPAD function to zero-pad numbers, or store the formatted numbers in a CHAR column.
```

虽然提示 ZEROFILL 快要被废弃了，但是为了演示区别，所以暂时还是不管了。
```sql
mysql> select * from test_int_zerofill;
+-------+--------+-------------+
| id    | no     | phone       |
+-------+--------+-------------+
|   010 |  00010 | 00000000010 |
| 12345 | 123456 | 00001234567 |
+-------+--------+-------------+
2 rows in set (0.00 sec)
```
所以对于问题1，所以 int(1) 和 int(10) 其实是没有区别的，唯一有区别的地方就在于如果使用了 `unsigned zerofill` 修饰的话，那么不足长度的会在左边进行补 0 ，而如果没有用 `unsigned zerofill` 进行修饰的话，可以说基本上是没有区别的。

对于问题2，因为 int 括号里面的值与位数无关， 所以是可以的。
到此，对于执行第一个SQL所进行的提示，是因为 Mysql 也觉得其实没什么意义，而是更加推荐用 `LDAP()` 这个函数来判断。

对于第二个问题，可以尝试一个手机号：
```sql
INSERT INTO `test`.`test_int_zerofill` (`id`, `no`, `phone`) VALUES (10, 10, 13100000000)

[22001][1264] Data truncation: Out of range value for column 'phone' at row 1
```
出现这个问题的原因在于 int 类型在 mysql 里面是四个字节，而一个字节是 8 位，所以在 mysql 里面，一个int 类型最多可以储存 2^31(一个符号位置)，也就是约 21 亿左右，无符号的也最多42亿，但是手机号最低也是11位，也就是 130 多亿。所以肯定是无法存储的。
下面是 Mysql 官方给出的 num 类型的存储范围
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%88%AA%E5%B1%8F2019-12-17%E4%B8%8A%E5%8D%8812.25.08.png)


## 源码
由于想看下 Mysql 到底是如何处理 int 类型的数值的，于是下载并且编译了 Mysql 的源码，一直跟着 debug，最后找到了 Mysql 判断 int 是否超长的一个代码，如下：
```C++
type_conversion_status Field_long::store(double nr) {
  ASSERT_COLUMN_MARKED_FOR_WRITE;
  type_conversion_status error = TYPE_OK;
  int32 res;
  nr = rint(nr);
  if (unsigned_flag) {
    if (nr < 0) {
      res = 0;
      error = TYPE_WARN_OUT_OF_RANGE;
    } else if (nr > (double)UINT_MAX32) {
      res = UINT_MAX32;
      set_warning(Sql_condition::SL_WARNING, ER_WARN_DATA_OUT_OF_RANGE, 1);
      error = TYPE_WARN_OUT_OF_RANGE;
    } else
      res = (int32)(ulong)nr;
  } else {
    if (nr < (double)INT_MIN32) {
      res = (int32)INT_MIN32;
      error = TYPE_WARN_OUT_OF_RANGE;
    } else if (nr > (double)INT_MAX32) {
      res = (int32)INT_MAX32;
      error = TYPE_WARN_OUT_OF_RANGE;
    } else
      res = (int32)(longlong)nr;
  }
  if (error)
    set_warning(Sql_condition::SL_WARNING, ER_WARN_DATA_OUT_OF_RANGE, 1);

#ifdef WORDS_BIGENDIAN
  if (table->s->db_low_byte_first) {
    int4store(ptr, res);
  } else
#endif
    longstore(ptr, res);
  return error;
}
```
在这段代码里面，Mysql首先会判断是否带有符号位，如果是无符号位的话，则是直接判断是否大于 UINT_MAX32，而如果是有符号的话，则是判断是否小于 INT_MIN32 或者大于 INT_MAX32，否则直接为最小或者最大值，然后设置error。
PS：这一段代码是还未进行 InnoDB 引擎层，可以看到 Mysql 是在 Server 层进行SQL语句的校验。