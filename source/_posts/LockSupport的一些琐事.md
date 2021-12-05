---
title: LockSupport的一些琐事
date: 2021-05-16 23:46:07
tags: [Java]
---
LockSupport 是一个用于线程阻塞或者唤醒的类，位于 rt.jar，主要是通过 Unsafe 类来进行操作

该类的方法都是静态方法，最常使用的方法是 void park(Object blocker) 和 void unpark(Thread thread)

是AQS操作的基础类，阻塞线程的时候不需要加锁，比较方便

下面主要介绍这两种方法：

## void park(Object blocker)
这个方法和 void park() 类似，最主要的区别在于 void park(Object blocker) 设置了一个 blocker，这个参数是一个 Object，一般是当前对象放进去，尤其是需要通过 jstack 查看线程阻塞的原因，此时会打印block
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/LockSupport_something/Jstack_Thread.png)

## void unpark(Thread thread)
该方法会将一个处于阻塞状态的线程唤醒，可以将「许可」理解为一个信号量
如果当前的没有信号量，则 unpark 会将设置一个，如果有一个信号量，则 unpark 也不会进行任何操作，因为LockSupport 所允许的信号量仅允许一个


## 特点
该类有一个特点就是，阻塞线程不需要在同步代码块里面
> 如果是 Object.wait()，需要在 synchronized 同步块里面的

唤醒一个由 park 方法阻塞的线程，被唤醒的线程是不知道由于何种原因导致的，例如唤醒它的方法有如下几种：
1. 另一个线程调用了 unpark 方法
2. 另一个线程调用了中断函数
3. 阻塞的时间到了，自动唤醒
被阻塞的线程可以由于上述任何一个原因被唤醒，但是唤醒之后如果要知道唤醒原因，除了第二种可以通过 `isInterrupted` 知道，其余两种均无法获知


## 使用注意事项
为了避免虚假唤醒，最好是在 while 里面调用 park 方法，同时 park 和 unpark 需要配对使用，但是需要注意不要在调用 park 方法之前调用 unpark 方法，避免导致 park 不会阻塞






