---
title: layui使用总结
date: 2018-03-22 18:24:47
tags: [web前端]
categories: [web前端,layui]
---

## 前言
由于自己平时对前端的css和js学的不是太好，而现在又需要自己来写一个前端页面，无意间在GitHub中看到了`layui`，所以抱着尝试的心态，学习了一下，现在主要是自己做一个总结。可能之后会学习Vue等前端跨框架

## 关于Layui的table组件
首先Layui的table组件：在Layui中创建一个table组件需要先写一个table标签：`<table class="layui-hide" id="test" lay-filter="demo"></table>`，在这之中 id 是需要在table.render中使用的。例如：
```
table.render({
            elem: '#test'
            ,height: 315
            ,url: '/user/getUser' //数据接口
            ,page: true //开启分页
            ,cols: [[ //表头
                {field: 'id', title: 'ID', width:80, sort: true, fixed: 'left'}
                ,{field: 'age', title: '年龄', width:80}
                ,{field: 'dataname', title: '用户名', width:80}
                ,{field: 'sex', title: '性别', width:80}
                ,{field: 'password', title: '密码', width:80, sort: true}
                ,{fixed: 'right', width: 165, align:'center', toolbar: '#barDemo'}
            ]]
        });
```
在这段js之中`elem`后面的值表示的就是table标签之后的id，而这段代码表示的就是在table中异步加载数据，后面的toolbar 表示的是需要创建三个button。
注意最后一行的`toolbar : #barDemo` 它会跟下面的三个按钮一起对应。
并且将这三个按钮一起添加到那个表格的后面，这之后的三个标签的创建方式如下:
```
<script type="text/html" id="barDemo">
        <a class="layui-btn layui-btn-primary layui-btn-xs" lay-event="detail">新增</a>
        <a class="layui-btn layui-btn-xs" lay-event="edit">编辑</a>
        <a class="layui-btn layui-btn-danger layui-btn-xs" lay-event="del">删除</a>
</script>
```
这三个<a>后的`lay-event`的值在之后会用到的。
```
table.on('tool(demo)', function(obj){ //注：tool是工具条事件名，demo是table原始容器的属性 lay-filter="对应的值" 
            var data = obj.data //获得当前行数据
                ,layEvent = obj.event; //获得 lay-event 对应的值
```
`function(obj)`的obj传入进来的是一些参数，obj.data 是获取所选中的数据行的一些值，obj.Event 获取的是前面的`lay-event`的值，在之后可以判断` if(layEvent === 'detail')`，然后就可以及进行操作了。 
```
layer.open({
                    type: 1,
                    title: "用户信息修改",
                    closeBtn: 1,
                    area: 'auto',
                    shadeClose: true,
                    skin: 'yourclass',
                    content: ''// 在这里可以写html标签然后会在那个面板显示了。 
})
```
```
layer.confirm('真的删除行么', function(index){
                    obj.del(); //删除对应行（tr）的DOM结构
                    //向服务端发送删除指令
                    layer.close(index);
                    $.ajax({
```
`confirm`可以弹出一个框来确认是否进行删除以免用户的误操作，而obj.del()则是可以删除该行。layer.close()则是可以关闭次对话框。
而`lay.message`则是一个弹窗，用于在页面上提示用户进行了什么操作。

## 和Jquery一起使用
这个UI框架是可以和Jquery一起来使用，Jquery可以发起异步请求从而来进行一些数据操作。



## 使用截图：
![](layui截图.jpg)