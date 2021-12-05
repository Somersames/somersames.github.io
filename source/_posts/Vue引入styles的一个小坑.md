---
title: Vue引入styles的一个小坑
date: 2018-10-13 22:50:59
tags: [web前端,css]
categories: [web前端,Vue]
---
在使用`styus`的时候，经常会定义一些css的常用变量，但是在引入styl文件一直使用的是`@import '~styles/mixins'`，然后就是一直在报错，
后来经过了解发现，这种写法需要在`build`中的`webpack.conf.js`中设置`style`的别名，

```js
resolve: {
    extensions: ['.js', '.vue', '.json'],
    alias: {
      'vue$': 'vue/dist/vue.esm.js',
      '@': resolve('src'),
      'styles': resolve('src/assets/styles')
    }
  },
```
然后将`styles`指向放`styl`的文件夹即可。