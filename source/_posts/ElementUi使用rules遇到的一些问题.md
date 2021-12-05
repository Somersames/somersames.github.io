---
title: ElementUI使用rules遇到的一些问题
date: 2019-05-30 00:07:43
tags: [Vue]
categories: [web前端,Vue]
---
这些天一直在踩`Vue`的坑...今天遇到的一个问题是在一个父组件中，将某些数据通过`props`传递给子组件，同时在子组件里面也有相应的一些`rules`规则，但是在实际的开发中，却发现子组件的`rules`并未生效...反而一直提示对应的 message，后来才发现是跟 ElementUI 的`prop`有关。

首先看一段代码:
```html
<template>
<div>
  <p>该Demo是为了测试Vue中rule和prop的不同</p>
  <el-form :model="vehicles">
    <el-form-item label="公共汽车车轮个数" prop="bus.wheel" :rules="{required: true,message: '请输入公共汽车车轮个数'}">
      <el-input v-model="vehicles.bus.wheel" placeholder="请输入"></el-input>
    </el-form-item>
    <el-form-item label="公共汽车司机驾照" prop="bus.driver.license" :rules="{validator: licenseCheck ,trigger:'blur'}">
      <el-input v-model="vehicles.bus.driver.license" placeholder="请输入"></el-input>
    </el-form-item>
    <el-form-item label="公共汽车车轮个数" prop="bus.driver.years">
      <el-input v-model="vehicles.bus.driver.years" placeholder="请输入"></el-input>
    </el-form-item>
     <car :car="vehicles.car"></car>
  </el-form>
</div>
</template>
<script>
import Car from './Car';

export default {
  components: {
    Car
  },
  data () {
    return {
      vehicles: {
        bus: {
          wheel: null,
          driver: {
            license: null,
            years: null
          }
        },
        car: {
          wheel: null,
          driver: {
            license: null,
            years: null
          }
        },
        train: {
          wheel: null,
          driver: {
            license: null,
            years: null
          }
        }
      },
      person: {
        child: {
          year: null
        }
      }
    };
  },
  methods: {
    licenseCheck (rule, value, callback) {
      let m = this.car;
      console.log(m);
      debugger;
      if (value != null) {
        if (value != 'A') {
          callback(new Error('必须A照'));
        } else {
          callback();
        }
      }
    },
    init () {
      console.log(this.car);
    }
  }
};
</script>
<style>

</style>

```
此时在这个组件中，一切都是正常的，但是一个完整的项目里面，是不可能将所有的元素都堆积在一个页面中，那样的话以后的维护就会非常的麻烦。所以此时就需要一个子组件，然后将父组件中一些数据传递至子组件。
代码如下:
```html
<template>
  <div>
      <p>小汽车子组件</p>
    <el-form-item label="小汽车车轮个数" prop="wheel" :rules="{required: true,message: '请输入公共汽车车轮个数'}">
      <el-input v-model="car.wheel" placeholder="请输入"></el-input>
    </el-form-item>
    <el-form-item label="小汽车司机驾照" prop="driver.license" :rules="{validator: licenseCheck ,trigger:'blur'}">
      <el-input v-model="car.driver.license" placeholder="请输入"></el-input>
    </el-form-item>
    <el-form-item label="小汽车车轮个数" prop="driver.years">
      <el-input v-model="car.driver.years" placeholder="请输入"></el-input>
    </el-form-item>
  </div>
</template>
<script>
export default {
  props: {
    car: {
      type: Object,
      default: function () {
        return {};
      }
    }
  },
  methods: {
    licenseCheck (rule, value, callback) {
      let m = this.car;
      console.log(m);
      debugger;
      if (value != null) {
        if (value <= 'D') {
          callback(new Error('必须C照以上'));
        } else {
          callback();
        }
      }
    },
    init () {
      console.log(this.car);
    }
  },
  mounted () {
    this.init();
  }
};
</script>
<style>

</style>

```
当然在这个页面里面，一切都是可以正常输入的...就是`rules`无法使用。由于自己才是刚刚开始接触`vue`和`ElementUI`，所以对`vue`里面的一些使用技巧还不是很熟悉，这个时候看了下父组件里面的`prop`和`v-model`，发现`prop`都是比`v-model`少一个前缀...所以以为在子组件里面也是这样..其实后来才发新这个。

然后再去查看`ElementUI`的官网，发现
> prop	表单域 model 字段，在使用 validate、resetFields 方法的情况下，该属性是必填的	string	传入 Form 组件的 model 中的字段

于时倒父组件中看了下，发现`el-form`确实有`:model`，然后参照了下`ElementUI`的介绍...突然想到是不是`prop`已经自动的将`:model`的对象带过来了...后来在子组件中进行了测试，修改后如下:
```html
<template>
  <div>
      <p>小汽车子组件</p>
    <el-form-item label="小汽车车轮个数" prop="car.wheel" :rules="{required: true,message: '请输入公共汽车车轮个数'}">
      <el-input v-model="car.wheel" placeholder="请输入"></el-input>
    </el-form-item>
    <el-form-item label="小汽车司机驾照" prop="car.driver.license" :rules="{validator: licenseCheck ,trigger:'blur'}">
      <el-input v-model="car.driver.license" placeholder="请输入"></el-input>
    </el-form-item>
    <el-form-item label="小汽车车轮个数" prop="car.driver.years">
      <el-input v-model="car.driver.years" placeholder="请输入"></el-input>
    </el-form-item>
  </div>
</template>
<script>
export default {
  props: {
    car: {
      type: Object,
      default: function () {
        return {};
      }
    }
  },
  methods: {
    licenseCheck (rule, value, callback) {
      let m = this.car;
      console.log(m);
      debugger;
      if (value != null) {
        if (value === 'A') {
          callback(new Error('必须A照'));
        } else {
          callback();
        }
      }
    },
    init () {
      console.log(this.car);
    }
  },
  mounted () {
    this.init();
  }
};
</script>
<style>

</style>

```

于是一切都正常了，后来为了测试是不是非要在`el-form`上加一个`:model`才能正常使用`rules`，所以就写了一个`el-form`测试。
```html
<template>
<div>
  <el-form>
    <el-form-item label="" prop="person.child.year" :rules="{validator: childCheck ,trigger:'blur'}">
      <el-input v-model="person.child.year" placeholder="请输入小孩的年龄" ></el-input>
    </el-form-item>
  </el-form>
</div>
</template>
<script>
import Car from './Car';

export default {
  components: {
    Car
  },
  data () {
    return {
      person: {
        child: {
          year: null
        }
      }
    };
  },
  methods: {
    childCheck (rule, value, callback) {
      debugger;
      if (parseInt(value) > 16) {
        callback(new Error('请输入16以下'));
      } else {
        callback();
      }
    },
    init () {
      console.log(this.car);
    }
  }
};
</script>
<style>

</style>

```

然后发现在`childCheck`里面`value`总是获取不到值...一直是 undefinded ,然后再在`el-form`里面加上一个`:modele`...修改如下:
```html
<template>
<div>
  <el-form :model="person">
    <el-form-item label="" prop="child.year" :rules="{validator: childCheck ,trigger:'blur'}">
      <el-input v-model="person.child.year" placeholder="请输入小孩的年龄" ></el-input>
    </el-form-item>
  </el-form>
</div>
</template>
<script>
import Car from './Car';

export default {
  components: {
    Car
  },
  data () {
    return {
      person: {
        child: {
          year: null
        }
      }
    };
  },
  methods: {
    childCheck (rule, value, callback) {
      debugger;
      if (parseInt(value) > 16) {
        callback(new Error('请输入16以下'));
      } else {
        callback();
      }
    },
    init () {
      console.log(this.car);
    }
  }
};
</script>
<style>

</style>

```

然后就都好了...

所以以后还是得多看看官方文档...