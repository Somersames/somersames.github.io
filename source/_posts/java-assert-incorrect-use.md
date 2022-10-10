---
title: 我就用了一个 assert，竟然让我损失了一杯星巴克
toc: true
date: 2022-09-07 20:09:05
tags: [Java]
categories: [Java,JVM]
---
虽然有那么点标题党，但是确实让我损失了一杯星巴克，耽误了测试小姐姐的时间～～

# 简介

事情的起因是这样的，我们有一个接口是对外修改状态用的，例如状态有1，10，20，30，40，50 等等，状态的流转在业务上来说只允许操作一次，不然会导致一些重复的事件触发。

所以为了防止并发修改导致的系统问题，我们在修改数据库的时候，做了一个 CAS 判断

```sql
update tbl set status = ? where status = ? and order_id = ? 

```


# 一）事件现场

在代码中，我们的 update 操作是会返回对应的影响的行数的，上面的 SQL，因为 order_id 是主键。所以在更新成功的时候，一定会返回 1。

于是我们的代码就是这样了：

```sql
assert assertTest.updateOrder() > 0;

```


然后呢，这代码在本地运行的非常符合要求，本地冒烟测试案例都是 100% 成功，于是我将这个代码提交到指定分支，然后就去弄别的去了。。

![image-20220910153637104](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202209111930825.png)

# 二）本地不是好好的吗

后来测试找过来了，后面有几个案例通过不了，接口返回更新成功，然后数据库实际却没变，还是原来的值，关键是还可以多次修改，数据库也没变…

此时我心里想？不应该啊，我在本地测试这么久，为啥发到测试环境就有问题🤔️

本着我本地是好的，因此我首先怀疑是测试操作不当。。于是亲自去看了一番操作。。没想到操作确实没问题。

## 2.1）难道是主从延迟

第一时间想到难道主从延迟，但是转念一想，就算从库同步主库的数据有问题，那么我的写操作全部是在主库，应该是 CAS 失败，不应该操作成功。

所以这个直接被否定了。

## 2.2）测试环境的包不对

我的第二感觉就是，难道测试环境的版本被其他人覆盖了？后来我看了下发版的 commitid，与我最后一次的提交对的上。

所以这个又被被否定了。

# 三）怀疑人生

此时这 case 直接弄得我怀疑人生。。于是一番冷静下来，首先查看日志请求，因为公司用的 cat，于是上去看了下，却发现 update 操作完全没有执行。

也就是 `assert assertTest.updateOrder() > 0;` 这一行一直没有执行。WTF，我本地不是好点吗？？

于是我又在本地自己 debug，发现一切正常，此时我就懵逼了，难道我用法不对。

我直接在项目中搜索了所有的 `assert` 关键字，发现 JDK 也是这样用的，感觉没多大毛病。

![image-20220910182733042](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202209111931261.png)

## 3.1）继续怀疑人生

既然 JDK 都这样在用，那么我这样也应该没什么问题吧，后来我转念一想，既然我遇到问题了，那么是不是其他人也会遇到问题。

直接上 stackoverflow 上去找找看。。还真找到了

![assert not working the way i thought in java](https://stackoverflow.com/questions/14026705/assert-not-working-the-way-i-thought-in-java)

这老哥代码和我一致，但是也不生效，他的代码如下：

```java
public void withdraw(double d){
  double diff = balance - d;
  assert (diff>=0 ) :" Insufficient funds!";
  balance = diff;
}
```



这个时候，有一个回答道：

![image-20220910183219120](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202209111931727.png)

看那个声望值，一看就是一个大佬，这个大佬回答道，assert 是默认被关闭的，需要通过 JVM 的配置来打开，于是我先看下我本地的 idea 配置，发现确实打开了

![image-20220910183411683](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202209111931078.png)

但是！！！测试环境的没打开。

# 四）解决方案

首先想到的是直接修改项目的 JVM 配置，但是考虑后续如果有人在跑我的代码的时候，也忘记打开这个开关了，估计会懵逼，所以索性直接批量替换了 `assert`，直接用 Spring 自带的 `Assert` 处理了。

# 五）后续

虽然问题解决了，但是我想了下，既然这个 `assert` 是依赖 JVM 配置的，那么如果不开启，是不是就代表着 JDK 所有的断言全部失败了？

然后我又抽查了几个 assert 关键字用的地方，发现大多数都是在 JavaFX 这个包里面，那么我就在好奇，为什么这个关键字大多数出现在 javafx 包里面。

在 StackOverFlow 上我看到了这个问题：
![assertions in JavaFx and in general](https://stackoverflow.com/questions/23528331/assertions-in-javafx-and-in-general)

![image-20220911191522514](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202209111931386.png)

提问者提到了一句话当 debug 的时候，非常的有用，我突然想到，我们现在用的各种软件，都有一个功能：*调试模式*

那是不是意味着 JavaFX 开发的软件，如果打开调试功能，就可以直接修改 JVM 配置，然后开启 `assert` 关键字。

这样 FX 内置的包就会通过断言将一些信息提前暴露出来。

# 六）后续

后续就是中秋节来，然后因为没有用过 JavaFX 开发软件，所以只能是自己猜测了
