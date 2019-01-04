---
title: SpringBoot中在一个事物中更新多表的注意事项
date: 2019-01-02 00:13:56
tags: [Springboot]
categories: Springboot
---
## 现象：
具体表现为数据被update之后，在同一个事物里面再次查询，查询的是一个更新之后的值。

## 复现步骤
更新一个商品的信息，其步骤如下：
1. 更新商品表的一些数据
2. 在进展表中新加入一条申请
3. 在操作日志表中新增各种变更的操作(`记录变更之前的值，变更之后的值`)

由于之前是每一个数据的操作都是独立的一个方法，所以其代码结构如下所示:
```java
@Transactional
public void ceateProductEditRequest(){
    public void editProduct();
    public void createNewProductProgress();
    public void insertLogs();
}
```

但是这样的操作顺序会出现一个问题，因为在同一个事物中。首先执行了更新商品的操作，然后在进展表中新增一个记录。一直到这里

在此之前一直都是没问题的，然而在第三步的时候，由于数据库中已经将商品表的数据进行了更新，所以此时查询出来的是一个更新之后的值。但是日志表中是需要记录申请的前后值得变化。所以此时就需要调整方法的顺序。


## 方案
同一个事物里面，在更新之后进行的查询语句，查询出的是更新之后的语句，如果要实现上述的业务场景，即需要将`insertLogs`方法提至`editProduct`之前。
```java
@Transactional
public void ceateProductEditRequest(){
    public void insertLogs();
    public void editProduct();
    public void createNewProductProgress();
}
```
由于在同一个事物里面，且该业务场景实际中并不多。所以这样简单处理了下。

但是这样的写法在并发高的情况下，需要考虑数据库的锁设计，防止出现了死锁，例如数据库开启了`GAP`锁或者`Next-Key Lock`