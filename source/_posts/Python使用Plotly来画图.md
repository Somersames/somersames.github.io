---
title: Python使用Plotly来画图
date: 2018-04-13 23:27:20
tags: [python,plotly]
---
Plotly是一个比较好的画图工具，主要用于数据的展示，以及分析，最好配合pandas一起使用发挥出最大的作用
## 引入基础库
```python
import plotly
import pymysql
import sys
import plotly.plotly
import plotly.graph_objs as go
import pandas as pd
```

## 画图：
在这里画图的话可以使用参照官网给出的一个饼形图例子：
```python
labels = ['Oxygen','Hydrogen','Carbon_Dioxide','Nitrogen']
values = [4500,2500,1053,500]
colors = ['#FEBFB3', '#E1396C', '#96D38C', '#D0F9B1']

trace = go.Pie(labels=labels, values=values,
               hoverinfo='label+percent', textinfo='value', 
               textfont=dict(size=20),
               marker=dict(colors=colors, 
                           line=dict(color='#000000', width=2)))
```
go.pie()函数的参数含义分别是名字，值，鼠标覆盖后需要显示的信息：标签和占比，最后一个是饼形图每一块显示内容，是限制数量还是什么，最后如果需要在本地会吐的话需要调用离线画图函数，但是貌似它的图形会比在线的少不少。

```python
   plotly.offline.plot([trace])
```
若需要绘制多个的话类似于折线图可以修改为`plotly.offline.plot([trace1,trace2])`或者直接`data=[trace1,trace2]`然后`plotly.offline.plot(data)`