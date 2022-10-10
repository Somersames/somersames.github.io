---
title: 用FastThreadLocal替代ThreadLocal
toc: true
date: 2022-10-10 20:05:53
tags: [Java]
categories: [Java]
---

在多线程编程的环境中，一个变量是可以被多个线程访问并修改的，如果想让一个线程，在不影响到其他线程的情况下，修改此变量，那么就需要将该变量改成自己私有的，这就是 ThreadLocal 的作用了。

# ThreadLocal

Threadlocal 可以将一个变量作为自己的私有变量，可以在本线程内随意修改并且不影响到其他线程，如下 Demo：

![image-20220724235813520](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202207252334607.png)

可以看到 t1 打印出 Thread1，t2 打印出 Thread2，两个线程的 ThreadLocal 都不受影响。

## 多个 ThreadLocal

在项目中，需要保存的值可能不仅仅一个，所有同一个 Thread 可以拥有多个 ThreadLocal，例如：

![image-20220725001506521](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202207252334833.png)

怎么实现的呢？在 JDK 中有一个 ThreadLocalMap，这个类的作用是将每一个ThreadLocal 作为 Key 保存于 Map 中，而 Map 的 value 就是我们在 ThreadLocal 中 set 的值了。

## ThreadLocalMap

在 Thread 中，有一个变量 threadLocals，其生命周期随着 Thread 的结束而结束。

![image-20220725001856312](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202207252334472.png)

### Map

如果大家的 JDK 八股文背的熟的话，应该可以记得在 Map 的接口中，有一个 Entry 的接口。

![image-20220725002106451](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202207252334570.png)

没错，就是这个，每一个实现 Map 接口类都必须实现这个 Entry

> 这里顺带提一句，接口里面是可以有内部接口的，跟内部类实现是一样的

然而 ThreadLocalMap 的 Entry 却跟这个没关系，在这里只是提一下这个

### ThreadLocalMap 的 Entry

![image-20220725002331080](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202207252335088.png)

ThreadLocalMap 的 Entry 是一个单独的类，同时是一个 WeakReference，关于 WeakReference，因为跟本文关系涉及关系不大，在这里就可以简单的理解为是下一次发生 GC 的时候，WeakReference 所持有的对象就会被回收了。

## ThreadLocal 进行新增

当我们向一个 ThreadLocal 进行赋值的时候，最终都会包装成一个 Entry，而 Key 就是 ThreadLocal，

![image-20220725003854729](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202207252335169.png)

所以 JDK 中的 ThreadLocal，其实就是作为一个 Key 存放于 ThreadLocalMap 中，而 ThreadLocal 的 value 就做为 ThreadLocalMap 的 value。

## 弊端

当 ThreadLocalMap 用 `Hash` 来计算下标的时候，就不可避免会带来 `Hash 碰撞`问题，在 ThreadLocalMap 中解决 `Hash`冲突的方法是 `线性探测` 法，即通过 Hash 计算下标，如果当前下标已经有值，那么就直接寻找下一个空下标，然后赋值。

如果在大量的碰撞，就会浪费大量的 CPU 资源。

# FastThreadLocal

这是 Netty 包里面的另一个 ThreadLocal 实现，它的功能和 ThreadLocal 类似，都是可以将一个变量设置为线程的副本，而装载 FastThreadLocal 的容器是 `InternalThreadLocalMap`。

Netty 中的实现，整体上和 JDK 中 ThreadLocal 没什么区别，但是具体细节上却是千差万别，先看 Netty 中的 InternalThreadLocalMap。

## InternalThreadLocalMap

这是由 netty 实现的一个 FastThreadLocal 的容器，类似于 JDK 中的 ThreadLocalMap，虽然是 Map，但是其内部是由数组来实现的。

![image-20220725005214905](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202207252335491.png)

UnpaddedInternalThreadLocalMap 是 InternalThreadLocalMap 的一个父类，定义了一些基本的变量。

这里的 `indexedVariables` 就是用来存放 FastThreadLocal 的下标，在 Netty 中，每一个 FastThreadLocal 都会有一个唯一的下标，从而在查找 value 的时候，直接通过下标在数组中进行定位。

而下标的产生就跟上图的 `nextIndex` 有关系了。

![image-20220725005635758](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/img/202207252335155.png)

创建一个 FastThreadLocal 的流程如下：

1. 通过 nextIndex 变量计算得到下一个下标
2. 如果当前下标小于 indexedVariables 的长度， 那么会进行扩容（一个损耗点）
3. 如果当前下标小于 indexedVariables 的长度，则直接进行赋值

## 开销

FastThreadLocal 的开销主要是在创建的时候分配 index 所产生的 CAS 竞争，后续就没有任何大的开销了，查询的时候也是直接通过下标就可以定位。

# 使用

对于一般的业务开发，ThreadLocal 已经够用了，没有必要更换为 FastThreadLocal，相比于 Hash 计算产生的那点影响，网络、IO 产生的性能消耗才是重点。

而 FastThreadLocal 的使用场景，一般事在一些需要处理大量数据的项目中，例如一个项目需要实时调用外部接口来获取数据，同时又用到了 ThreadLocal 来进行存储一些基本的信息。

那么这个时候是可以考虑将 ThreadLocal 替换为 FastThreadLocal，首先是这种项目 GC 非常的频繁，一旦处理不好就会导致 Entry 中的 ThreadLocal 被回收，其次这种项目大部分会用 Netty 来进行网络传输，自带 FastThreadocal，不会担心额外的引入第三方包产生的风险。

# 杂谈

## 内存泄漏

在使用 JDK 自带的 ThreadLocal 的时候，使用不当可能会产生内存泄漏：

1. 因 GC 导致 Entry 的 ThreadLocal 被回收
2. 线程长时间的运行，并且 ThreadLocal 没有被及时的 remove

### 因 GC 导致 Entry 中的 ThreadLocal 被回收

主要原因是 ThreadLocalMap 的生命周期和 Thread 是一致的，而 ThreadLocalMap 中的 Entry 的 Key 是 WeadReference，一旦发生GC，Key 可能被直接释放，但是 value 因为一直被 Entry 引用，所以导致无法被及时释放。

### 线程长时间的运行，并且 ThreadLocal 没有被及时的 Remove

Thread 在使用完以后没有及时的 remove ThreadLocal，这种情况多发生于线程池的使用，如果项目中有一个通用线程池，有人在使用以后忘记进行将本次任务的 ThreadLocal 进行 remove，就会导致这部分的 ThreadLocal 也无法释放，引发内存泄漏。

## FastThreadLocal

FastThreadLocal 的做法是直接将 Object 数据强引用，所以除非是真正的内存不足，否则 FastThreadLocal 中的值不会被回收。

并且 FastThreadlocal 在每一次线程执行完毕以后，都会主动的进行 remove 操作，可以避免第二点导致的内存泄漏，但是对于第一点需要开发人员注意。
