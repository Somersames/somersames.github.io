---
title: css中定位的学习
date: 2018-10-13 23:23:38
tags: [web前端,css]
categories: [web前端,css]
---


首先页面代码如下：
```html
    <div id='div1'>
        <div id='div2'>
            <div id='div3'>
                <div id='div4'>

                </div>

            </div>

        </div>
        <div id='div22'>
        </div>
    </div>

    <style>
    #div1{
        width: 700px;
        height: 700px;
        background: red;
        margin-top: 50px;
        margin-left: 50px;
    }
    </style>
```
# relative和absolute
relative单独使用，代码如下
```css
#div2{
        width: 500px;
        height: 500px;
        background: rgb(17, 0, 255);
        position: relative;
        top: 20px;
        left: 20px;
        right: 0;
        bottom: 20px;
    }
}
```
其显示结果如图：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/relative-1.png)



说明一切都是没问题的，div2是按照div1来进行相对定位的。那么单独使用`absolute`呢
# absolute单独使用
```css
 #div2{
        width: 500px;
        height: 500px;
        background: rgb(17, 0, 255);
        position: absolute;
        top: 20px;
        left: 20px;
        right: 0;
        bottom: 20px;
    }
```

![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/absolutu.png)

可以看到使用`absolute`的`div2`并未按照`div1`

那如果`div3`也是absolute定位的话，那么此时会相对于`div2`定位还是相对于`div1`呢?
测试如下：
```css
#div1{
        width: 700px;
        height: 700px;
        background: red;
        margin-top: 50px;
        margin-left: 50px;
    }
    #div2{
        width: 500px;
        height: 500px;
        background: rgb(17, 0, 255);
        position: absolute;
        top: 20px;
        left: 20px;
        right: 0;
        bottom: 20px;
    }
    #div3{
        width: 300px;
        height: 300px;
        background: green;
        position: absolute;
        left: 10px;
        top: 10px;
    }
```
此时页面的显示如下：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/absolute-2.png)
也就是说，`div3`是按照`div2`来定位的，那么为什么就是`div2`是按照html定位的呢？
解释就是，使用`absolute`定位的元素会一直向上寻找，直到找出包含`position`的一个元素，然后按照其定位，那么`relative`呢?
```css
#div1{
        width: 700px;
        height: 700px;
        background: red;
        margin-top: 50px;
        margin-left: 50px;
    }
    #div2{
        width: 500px;
        height: 500px;
        background: rgb(17, 0, 255);
        position: relative;
        top: 20px;
        left: 20px;
        right: 0;
        bottom: 20px;
    }
    #div3{
        width: 300px;
        height: 300px;
        background: green;
        position: relative;
        left: 10px;
        top: 10px;
    }
```
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/relative-2.png)

可以看到，其定位都是直接以其上级元素为标准的。


# absolute的特性
absolute 具有的特性之一就是其包裹性，也就是`absolute`的`width`的宽度是100%的时候，其宽度其实是内容的宽度，而不是真正的`100%`宽度。即被`inline-block`化了。
## 妙用1
```html
<div id='div-span-1'><span id='span-1'>adad</span></div>
<style>
 #div-span-1{
        /* width: 700px; */
        /* height: 200px; */
        position: absolute;
        color: pink;
        border: 10px solid black
    }
    /* #span-1{
        position: absolute;
    } */
</style>
```
注意此时上层div不可以设置width，这样就可以实现外层div的宽度自动是内容的宽度
修改为如下，也可以使用：
```css
 #div-span-1{
        /* width: 700px; */
        /* height: 200px; */
        position: absolute;
        color: pink;
        border: 10px solid black;
        float: left;
    }
```
## absolute会破坏其父元素的宽度：
```html
 <div id='div-span-1'><span id='span-1'></span></div>
 <style>
 #div-span-1{
        /* width: 700px; */
        /* height: 200px; */
        /* position: absolute; */
        color: pink;
        border: 10px solid black;
        float: left;
    }
    #span-1{
        width: 300px;
        height: 300px;
        position: absolute;
        border: 10px solid gray
    }</style>
```
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/absolute-3.png)

此时可以看到父元素的高度已经被破坏了