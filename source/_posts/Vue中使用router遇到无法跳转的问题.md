---
title: Vue中使用router遇到无法跳转的问题
date: 2018-10-31 00:14:20
tags: [web前端,Vue]
categories: [web前端,Vue]
---

今天在配置Vue的路由的时候，在router的`index.js`中配置了如下代码：
```js
import Vue from 'vue'
import Router from 'vue-router'
import Home from '@/pages/components/home/Home'
import Tree from '@/pages/components/tree/Tree'

Vue.use(Router)

export default new Router({
  routes: [
    {
      path: '/',
      name: 'Home',
      component: Home
    },
    {
      path: '/tree',
      name: 'Tree',
      component: Tree
    }
  ]
})
```
但是却在启动项目的时候，输入URL:`localhost:8080/tree`无法实现跳转，后来发现是在`App.vue`中没有设置`<router-view>`,最后加上去即可。

例如在`App.vue`中配置了一个`<router-view>`，输入URL:`localhost:8080/tree`就会跳转到了`Tree.vue`。
那假设在`Tree.vue`中再配置配置一个`<router-view>`，这时候就会匹配的是`localhost:8080/tree/XXX`,当然也可以直接用`<router-link>`跳转到指定的组件中