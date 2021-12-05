---
title: 在Vue中使用filters来进行字典值的转换
date: 2019-05-27 23:07:10
tags: [Vue]
categories: [web前端,Vue]
---

在Vue里面，经常会遇到一些字典值的转换，而这些字典值由于和后端进行了约定的，一般不会轻易的改变，所以在前后端开发的项目中，这种字典值最好的做法是前端独立的保存一份，自己在前端自行进行处理。

我们的做法是使用`Vuex`的`store`配合`filters`来进行前端的字典值转化，首先是在`store`里面将字典值进行固定，然后通过`filters`在页面中进行一个转换。


## 使用Store
在`Vue`里面使用`store`首先需要安装`vuex`，安装完毕之后就可以直接在`main.js`里面直接引用了，但是为了统一管理还是决定新建一个`store`文件夹，然后将`store`相关的文件全部统一存放，新建完毕之后项目结构如下：
```js
---App.vue
---main.js
---store
-----index.js
```
在新建的`index.js`里面将`Vuex`实例注入到`Vue`中，如下:
```js
import Vue from 'vue';
import Vuex from 'vuex';
Vue.use(Vuex);
const store = new Vuex.Store({
  state: {
    score: [
      60,
      80,
      100
    ],
    enum: {
      60: '及格',
      80: '良好',
      100: '优秀'
    }
  }
});

export default store;
```
在这里我定义了两个变量，一个是`score`，一个是`enum`，`score`主要是为了展示一些固定的值在前端的展示，而`enum`则是准备介绍`filter`的使用

### 新建一个Vue页面
在这个页面里面，主要是介绍`store`的直接使用

```html
<template>
<div>
  <div v-for="(item, index) of score" :key="index">
    {{item}}
  </div>
</div>
</template>
<script>
import {mapState} from 'vuex';
import Vue from 'vue';

export default {
  computed: {
    ...mapState(['score'])
  },
  }
};
</script>
<style>

</style>

```
然后查看页面，就会发现页面上已经出现了三个分数，分别是在`store`里面定义的60，80，100。这种方式是通过`...mapState`来获取的`state`里面定义的一些值。

通过这种方式有几种好处，第一就是当需要改变前端某一个字段的值的时候，则可以直接通过`store`从而减少对项目的改动，其二就是可以让前端项目更加规范、可扩展。


## 使用filter来进行一些值的处理
为了大大提高前端的可扩展性，通过会对一些固定的值进行转换。例如性别，后端可能会返回 0 或者 1，若前端在某些页面上需要显示为`男|女`，而在某一些页面上需要显示`先生|女士`，此时通过`filter`来进行处理，则是一个不错的选择。
### 新建一个文件夹和js文件
```js
---App.vue
---main.js
---store
-----index.js
---utils
-----filter.js
```

filter.js
```js
import store from '../store/index';

export function enumConvert (val) {
  return store.state.enum[val];
}

```

main.js
```js
import * as filters from './utils/filter';

Object.keys(filters).forEach((key) => {
  Vue.filter(key, filters[key]);
});
```

在这里通过`Vue.filter`将filter方法进行全局注册，然后在`Vue`页面进行使用
```html
<template>
<div>
  <div v-for="(item, index) of score" :key="index">
    {{item | convert}}
  </div>
</div>
</template>
<script>
import {mapState} from 'vuex';
import Vue from 'vue';

export default {
  computed: {
    ...mapState(['score'])
  },
  filters: {
    convert (val) {
      return Vue.filter('enumConvert')(val);
    }
  }
};
</script>
<style>

</style>

```
此时页面上就会展示`及格`，`良好`，`优秀`了