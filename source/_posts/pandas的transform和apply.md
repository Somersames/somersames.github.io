---
title: pandas的transform和apply
date: 2017-12-10 16:22:20
tags:
---

# q区别
transform是Pandas里面Groupby的一个方法，主要作用是对groupby之后的dataframe进行处理，接收的参数一个是一个Series
```python
In [18]: df = pd.DataFrame({'B': ['one', 'one', 'two', 'three','two', 'two', 'one', 'three'],
    ...:                    'C': [1,2,3,4,5,6,7,8], 'D': [11,12,13,14,15,16,17,18]})

In [19]: df
Out[19]: 
       B  C   D
0    one  1  11
1    one  2  12
2    two  3  13
3  three  4  14
4    two  5  15
5    two  6  16
6    one  7  17
7  three  8  18

```
那么需要对其groupby之后求C的平均值怎么办
```
n [23]: df.groupby('B').transform(lambda x : x.mean())
Out[23]: 
   C   D
0  3  13
1  3  13
2  4  14
3  6  16
4  4  14
5  4  14
6  3  13
7  6  16

In [24]: df.groupby('B').apply(lambda x : x.mean())
Out[24]: 
              C          D
B
one    3.333333  13.333333
three  6.000000  16.000000
two    4.666667  14.666667

```
同样是一个lambda表达式，那么为什么会出现两种不同的结果：
```
In [40]: df1 = pd.DataFrame({'B': ['one', 'two', 'three','four'],
    ...:                    'C': [1,2,1,1], 'D': [11,12,13,14]})

In [41]: df1
Out[41]: 
       B  C   D
0    one  1  11
1    two  2  12
2  three  1  13
3   four  1  14

In [42]: def fun1(x):
    ...:     print x
    ...: 

In [43]: print df1.groupby('C').transform(fun1)
0      one
2    three
3     four
Name: B, dtype: object
0    11
2    13
3    14
Name: D, dtype: object
       B   D
0    one  11
2  three  13
3   four  14

1    two
Name: B, dtype: object
1    12
Name: D, dtype: object
       B   D
0    one  11
1    two  12
2  three  13
3   four  14

```
从上面可以发现trandform每次传入的是一个series，即按照C分组之后，首先传入的是‘B’列，然后是‘D’列，然后是'B''D'列，那么对于apply来说他传入的是什么呢：
```
In [45]: df1.groupby('C').apply(fun1)
       B  C   D
0    one  1  11
2  three  1  13
3   four  1  14
       B  C   D
0    one  1  11
2  three  1  13
3   four  1  14
     B  C   D
1  two  2  12
Out[45]: 
Empty DataFrame
Columns: []
Index: []
```
打印结果如上，说明apply和transform在接受参数上的差异(一个接受的是DataFrame，一个是Series)，上图中的打印第一个和第二个重复问题待会再解释，
那么假设有如下的lambda表达式：
> lambda x : return ['C']-x['D'] 
结果会怎样显示? 新增加一列：
```
In [49]: df1['E']=20
In [50]: df1
Out[50]: 
       B  C   D   E
0    one  1  11  20
1    two  2  12  20
2  three  1  13  20
3   four  1  14  20
In [51]: df1.groupby('C').transform(lambda x :(x['D']-x['E']))
Out[51]: 
       B   D   E
0    one  11  20
1    two  12  20
2  three  13  20
3   four  14  20

```
那么出现这样的情况的原因是什么呢？如前面所示的，transform传入的是一个series