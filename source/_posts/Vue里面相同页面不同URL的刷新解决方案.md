---
title: Vue里面相同页面不同URL的刷新解决方案
date: 2019-07-24 00:09:58
tags: [Vue]
categories: [web前端,Vue]
---
随着项目的收尾，使用`Vue`也有大概两个月时间了，在这期间也才遇到过了不少的问题，今天就来说下 `Vue` 中URL不同但是页面相同的解决办法。

## 现象

假设有两个URL，都对应的是一个相同的 Vue 页面，URL分别是 `view/account/1` 和 `edit/account/1`，此时如果由 `view/account/1` 跳转至 `edit/account/1`，你会发现页面是不会刷新的， 从而直接影响了整体的功能。

于是去网上寻找解决方案，在一个 github 的 issue 里面，看到也有人反映过这个问题，不过作者的回复是采用

`reload` 方式进行强制刷新，也就是类似于 F5 那样，页面首先会白一下，然后就再出现元素。这样虽然可以实现页面上元素的一些加载，但是同时它的弊端也体现出来了，就是对用户极度的不友好，所以最后我们才用了一种Reload的方式来进行刷新的。

首先是在`App.vue`里面添加如下代码：

```js
<template>
  <div id="app">
    <router-view v-if='isAlive'/>
  </div>
</template>

<script>
export default {
  name: 'App',
  provide () {
    return {
      reload: this.reload
    };
  },
  data () {
    return {
      isAlive: true
    };
  },
  methods: {
    reload () {
      this.isAlive = false;
      this.$nextTick(function () {
        this.isAlive = true;
      });
    }
  }
};
</script>

<style>
</style>

```



然后再需要刷新的 Vue 页面直接通过 `inject` 来进行使用即可：

```js
 <template>
    <div>
      <el-form :model="account">
        <el-form-item label="姓名" prop="name" >
          <el-input v-model="account.name" placeholder="请输入"></el-input>
        </el-form-item>
          <el-form-item label="年龄" prop="age" >
          <el-input v-model="account.age" placeholder="请输入"></el-input>
        </el-form-item>
          <el-form-item label="性别" prop="gender" >
          <el-input v-model="account.gender" placeholder="请输入"></el-input>
        </el-form-item>
      </el-form>
    </div>
</template>
<script>
export default {
  inject: ['reload'],
  data () {
    return {
      account: {
        name: null,
        age: null,
        gender: null
      }
    };
  },
  watch: {
    '$route' (to, from) {
      this.reload();
    }
  }
};
</script>
<style>

</style>

```

最后便可以实现页面状态的重新加载。

