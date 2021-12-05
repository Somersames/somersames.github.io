---
title: 记一次sql嵌套查询的使用方法
date: 2018-03-16 20:57:29
tags: [MySql]
categories: [数据库,MySql]
---

# 记一次sql嵌套查询的使用方法
一般来讲在sql中嵌套查询在where之后以便于查询范围的限制。
现在有一个情况就是就是现在嵌套查询的话是需要查询出结果值然后返回为一个字段。

## 思路：
#### 思路一：既然需要返回的是一个字段，那么是需要一个嵌套查询，所以一般的表达式是：
`select <表达式>（select<表达式>）as name`这样作为一个查询。
首先在括号`()`里面的一个select语句是可以作为一个字段的，可以通过在后面加一个as `字段`从而返回的是一个字段。、
那么这样书写sql之后便可以作为一个字段然后返回结果了。但是这样写有一个弊端，就是若没有groupby，则会导致查询出来的数据会有多余重复得。
类似下面这条sql：
```
SELECT * ,(SELECT SUM(CASE WHEN car_information.`car_status` ='1' THEN 1 ELSE 0 END)
FROM car_information WHERE car_information.`cat_place`='武汉') AS 'weifacishi' 
FROM car LEFT JOIN car_information ON car.`car_num`=car_information.`car_num` ;  
```
查询结果：

![](sql重复数据.png)

可以看到重复了许多数据，所以表明需要实现这个查询集合需要修改sql语句。
#### 思路二：

当看到这种写法出现重复得时候，便决定这个sql必须进行groupby，首先写了一个sql来测试groupby得，发现可以。
```
SELECT car_information.`car_num`,car_information.`cat_place`, SUM(CASE WHEN car_information.`car_status` = '1' THEN  1  ELSE 0 END) AS 'weifacishi'
 FROM car_information GROUP BY car_information.`car_num`;
```
查询结果：

![](进行Groupby测试.png)
发现，思路确实是对的，只不过需要对sql重新进行排序。

#### 思路三：
有了上面得两步之后便差不多知道如何进行操作了：首先应该进行一个左连接，以car表为左表，car_information为右表。连接之后再这个结果集中进行groupby然后统计。

```
SELECT * FROM (SELECT car.`car_num`,car.`car_wearhouse`,car_information.`car_status` FROM car LEFT JOIN car_information ON car.`car_num`=car_information.`car_num`) AS t GROUP BY t.car_num;
```
结果如下：

![](正确得结果.png)

## 总结：
1. 在sql语句中，几个表之间得连接是可以做一个结果集得，这个结果集可以通过as命名为别名然后通过操作这个别名来操作；

2. 不理解得地方：
    在sql中`select t.XXX`这个t是后来才被命名得，但是在之前却是可以使用得，因此想知道sql得运行原理---以后再了解

3. 在sql中统计得用法`case when XX=='XX' then 1 else 0 end`，并且嵌套得使用得() as 'XXX'
## 其余的以后再补充

#### 补充一：
在sql中左连接和内连接的优先选择问题，内连接会返回在两表中都包含的值，而左连接的话，当右边的表数据在左表中不存在的时候会出现null，而内连接则不会产生这种情况。
还有就是UNION的使用，union在两张表返回的结果集是一样的话而且两表需要拼接的话是非常有效的