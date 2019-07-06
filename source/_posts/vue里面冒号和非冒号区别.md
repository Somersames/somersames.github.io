---
title: vue里面冒号和非冒号区别
date: 2019-06-27 00:07:16
tags: [Vue]
categories: Vue
---
## vue里面冒号和非冒号的却别

今天在使用Vue的时候，突然发现了一个问题，就是在后端传过来的值因为是一个`boolean`类型的，但是前端又需要进行展示，由于我们使用的是`ElementUI`的话，于是参照官网上就是这样写的：

```vue
<el-select v-model="option_boolean">
        <el-option label="1" value="true"></el-option>
        <el-option label="2" value="false"></el-option>
</el-select>
```

但是此时页面上展示并非是 1 和 2 ，而是 true 和 false。如下：

![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%9C%89%E9%81%93%E4%BA%91%E7%AC%94%E8%AE%B0/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202019-06-26%20%E4%B8%8B%E5%8D%8810.20.15.png)

按照正常的逻辑，此时这个下拉框里面的值应该是1，而不是true。如果此时修改为如下写法：

```vue
<el-select v-model="option_boolean">
        <el-option label="1" :value="true"></el-option>
        <el-option label="2" :value="false"></el-option>
    </el-select>
```

![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202019-06-26%20%E4%B8%8B%E5%8D%8810.22.58.png)

此时的页面就会显示正常了。





## 解释

在 Vue 里面，冒号`:`代表的是一个双向绑定，其值要么是一个变量，要么是一个函数，而此 Demo 里面，第一个例子中，value仅仅是作为一个属性，所以它只能是接受字符串类型等

但是在第二个例子里面，由于使用了 Vue 的一个双向绑定模式，所以此时便可以正确的识别出 boolean 类型的值了