---
title: Vue中nextTick的使用
date: 2019-05-20 23:39:42
tags: Vue
categories: Vue
---

这段时间一直在负责一个前后端项目，前端是`Vue`+`ElemeUI`，由于自己之前只是会简单的使用 Vue 的一些初级命令，自然只能慢慢的踩坑，然后再出坑....。例如`数组无法触发Vue的视图更新`

刚开始在使用 Vue 的时候，一直都是在 `created` 方法里面获取后端数据进行渲染，这样用起来倒也没什么问题，只不过今天突然看到了 Vue 的`nextTick` 方法，感觉比之前在`created`里面请求后端更加高级。所以顺便研究了一波。

## 使用场景
`nextTick`常用于数据更新后，但是dom元素还未完成刷新，如何理解呢? 在 Vue 里面，更新 DOM 元素是异步的，也就是说当我们修改了数据之后，DOM元素并不会立即被刷新。参考[深入响应式原理](https://cn.vuejs.org/v2/guide/reactivity.html#%E5%BC%82%E6%AD%A5%E6%9B%B4%E6%96%B0%E9%98%9F%E5%88%97)

如下Demo
```html
<template>
<div>
<div :id='id'></div>
<el-button type="button" @click="nextClick">点击测试</el-button>
</div>
</template>
<script>
export default {
  data () {
    return {
      id: 'q'
    };
  },
  mounted () {
  },
  methods: {
    nextClick () {
      this.id = 'm';
      let obj = document.getElementById(this.id);
      var _this = this;
      this.$nextTick(() => {
        let one = document.getElementById(_this.id);
        console.log(one);
      });
      console.log(obj);
    }
  }
};
</script>
```
此时你在控制台会看到`obj`获取的是null，`one`获取的dom节点才是正确的。


[StackOverFlow关于这个的另一个解释](https://stackoverflow.com/questions/47634258/what-is-nexttick-or-what-does-it-do-in-vuejs)