---
title: Python2.7使用Pandas连接数据库
date: 2018-04-11 23:06:57
tags: [pandas,Mysql]
categories: Python
---
今天遇到一个需求，需要将Excel中的一些数据导入到mysql中，由于之前接触到了Python的Pandas，所以这个时候便想到了Python，但是连接数据库的时候出现了问题，所以便写一个文章记录下。

## 解决办法：
1. 下载Mysql_Python的一个exe文件
2.  注意`tosql`的这个方法使用的类。pd.io.sql.to_sql
3.  注意添加`index=False`防止出现出入的时候多了一个index

## sqlalchemy方式连接

### 导入库
由于使用的版本是Python2.7.14，所以在安装`MySQLdb`的时候一直出现问题，大意就是说需要升级pip，但是pip已经升级了。所以去网上查询解决办法是需要安装一个文件`Mysql-Python。。。.exe`但是这里需要注意版本，官网下载的是win32的，所以导致一直识别不到Python2.7的路径，导致`MySQLdb`这个一直安装不上。

另外需要导入的库就是Pandas了

### to_sql()
在查看官网API(pandas1.6.0版本)的时候，发现是pd.to_sql()，但是实际上这里是`pd.io.sql.to_sql()`,参数还是不变。
```python
    def ins(cls):
        xls = pd.ExcelFile(U'C:\\Users\\SZH\\Desktop\\aaa\\product表.xlsx')
        previous_score_csv_file = xls.parse(U'Sheet1');
        conn = create_engine('mysql+mysqldb://root:root//@localhost:3306/vendor?charset=utf8')
        pd.io.sql.to_sql(previous_score_csv_file,"product",conn,if_exists='append',index=False)
        # c.commit()
        print  previous_score_csv_file
```
最后便可以了。


## MySQLdb方式连接

暂时找不到解决办法

