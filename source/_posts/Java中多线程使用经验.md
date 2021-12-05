---
title: Java中多线程使用经验
date: 2021-10-21 00:14:30
tags: [并发]
categories: Java
---
# 在子线程中获取父线程的ThreadLocal
如果想子线程想使用父线程的 ThreadLocal，那么父线程中的 inheritableThreadLocals 有值，这样子线程中的 init 方法就会自动的将父线程的 inheritableThreadLocals 设置为 ThreadLocal
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/Java%E4%B8%AD%E5%A4%9A%E7%BA%BF%E7%A8%8B%E4%BD%BF%E7%94%A8%E7%BB%8F%E9%AA%8C/clipboard_20211101_125819.png)

但是因为子线程是在 init 方法中进行赋值的，所以如果子线程是由线程池创建的，那么该方法就又可能会失效，当线程池刚初始化完毕的时候，此时线程池中的还没有线程，当调用  `execute` 方法，此时就会 new 一个线程，那么这时候子线程是可以读取到父线程中 inheritableThreadLocals 的值

但是线程池中的线程是可以被复用的，所以后续如果线程不再创建的时候，那么子线程便不能再次获取父线程中的 inheritableThreadLocals，也就无法再将 ThreadLocal 进行父子线程的传递

# 如何感知到子线程的异常
这种业务场景常见于一定时任务，尤其是一些需要多线程并发处理的job，理想情况是所有的job都执行成功了，但是如果有异常情况，那么需要及时的通知开发人员查看日志。
从代码的可维护性上来说，每一个单独的子job最好只处理本job该处理的事情，而一些告警通知或者重试机制，则留给上层来处理。

实现的方式有两种：
1. 子job抛出异常，然后父线程通过 `future.get()` 来获取子线程的异常信息
2. 子线程抛出一个特定的异常，父线程感知到以后通知开发，并且决定业务的接下来走向

第一种的方式实现上比较繁琐，而且代码非常不美观，因此不太建议，当然如果多线程执行的job最后需要在主线程进行聚合操作，那么第一种方式还是可取的


下面来说下第二种，设置 `UncaughtExceptionHandler`，这种既可以为单独一个线程设置指定异常的抛出

也可以为一个线程组来设置，当线程没有设置的时候，会判断当前线程组是否有线程异常处理器

所以第二种方式其实也适用于线程池，线程池中有一个参数是 ThreadFactory，而这个类里面有一个参数是 ThreadGroup，因此在创建线程池的时候，如果自定义一个 ThreadFactory + ThreadGroup，那么是完全可以实现线程池中的线程异常统一处理

http://www.codebaoku.com/it-java/it-java-225099.html



