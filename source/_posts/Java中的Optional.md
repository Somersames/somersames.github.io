---
title: Java中的Optional
toc: true
date: 2022-07-08 20:01:40
tags:
---
# Java中的Optional

目前的项目中，JDK 版本已经全部是 1.8 以上了，项目函数式编程的代码多了起来，但是由于每个人的语言风格不同，如果滥用 JDK8 的各种函数式 API，会造成一定的坏味道

# Optional

网上最多的说法是为了避免 NPE（NullPointException），但是我个人觉得这和说法不太成立，例如下面的一个方法：

![image-20220703175042621](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202207080015855.png)

如果调用方不注意进行非空判断，那么一样会导致 NPE，所以在判断某一个变量是否为空的这一点上，其实 Optional 和 != null 没有本质上的区别。

Optional 的强大之处在与后续的 API

![image-20220703175350144](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202207080015240.png)

下面举一个例子：

如果你需要找出一个用户的子账户里面的家庭地址，如果不使用 Optional，可能是这种写法：

```java
if (user != null) {
    Account subAccount = user.getSubAccount();
    if (subAccount != null) {
        Address address = subAccount.getAddress();
        if (address != null) {
            return address.getDetail();
        }
    }
}
```

这种子弹型写法非常的不雅，而且后续也非常不利于维护，如果换成 Optional 的写法就简单多了。

```java
String result = String address = Optional.ofNullable(u)
                .map(User::getSubAccount)
                .map(Account::getAddress)
                .map(Address::getDetail)
                .orElse("");
```

Optional 会自动的判断当前 Stream 中的元素是不是为空，如果为空就终止，并且将 `orElse` 的值进行返回，在一定的程序上可以预防 NPE 的出现。

> 如果正常的代码中，有人忘记判断非空，例如 address 未判空就去取 detail 的值，就会导致了 NPE

业务实践上，到底方法的返回需不需要加 Optional，这个确实就见仁见智了，加 Optional，如果调用方直接 `get` 进行使用，IDEA 会显示黄色的 warning，在条件允许的情况下，再配合一个自动化的代码检测，那么可以很大程度上减少生产事故的发生。

但是这样导致的一个弊端就是代码的可读性降低，任何一个取值的地址，都必须调用 `isPresent`然后 `get()`，还不如 != null 来的实在。

所以个人建议是底层的方法不返回 Optional，最基础和最核心的方法，就返回对象或者报错就行了，业务的组合逻辑可以随意使用。

## 坑

再说下团队在使用 Optioal 的一些注意事项。

### orElse
![image-20220704002158449](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202207080015508.png)

这个方法并不是说只有第一个为 null，第二个才会执行，而是无论第一个有不有值，第二个都会执行，所以打印的结果就是

```java
getA
getB
1
```


如果想实现第一个为null，就执行后面的方法，否则不执行的这个逻辑，就需要这个方法了。

## orElseGet
![image-20220704002529580](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202207080015628.png)

此时如果 getA 返回的有结果，则 getB不会执行。

## ifPresent

这个方法是我用的最多的一个方法，主要是用来代替下面的代码：

```java
if(XXX ! = null){
  YYY
}
```

这种就可以优化为：

```java
Optional.ofNullable(XXX).ifPresent(YYY)

```

非常的方便。

# 最后

Optional 其实是比较基础的一个类，JDK8 中最重要的还是 Stream，如果喜欢函数式编程，可以看下 vavr
