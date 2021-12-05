---
title: CountDownLatch详解
toc: true
date: 2021-12-04 23:50:12
tags: [Java]
categories: [Java,JDK]
---
# CountDownLatch

CountDownLatch 只有一个构造函数：`CountDownLatch(int count)`

其中 count 表示该信号量的数量，其中具体的实现类是 Sync，而 Sycn 又是继承自 AQS，实现了几个 AQS 的方法

在生产环境中可以用于分治思想，讲一些复杂的处理分成一些子任务，等所有处理任务处理完毕以后，主线程才会执行

还可以用于一些任务的流程检查，例如只有所有的检查都完毕以后，主线程才可以获取数据然后执行



具体的实现，是利用了 AQS 的一个 volatile 变量 state

## countDown

该方法的具体流程就是将 state 通过 CAS 操作进行原子性的 -1，然后判断 -1 后的 state 是不是变成了 0 ，如果是的话，则会调用 doReleaseShared 方法唤醒后续的一个节点。

关于 doReleaseShared，因为是 AQS 来负责具体实现的，所以在这里先不做说明，在文章后半部分会进行说明

![image-20211204235609164](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/CountDownLatch%E8%AF%A6%E8%A7%A3/image-20211204235609164.png)

## await

如果调用 await 的时候，state 已经是0了，那么此时就会直接返回，当前线程也不会被挂起，而如果 state 不是0，那么当前线程就会被阻塞

如果调用该方法的时候，线程已经被设置了中断，那么会直接抛出  InterruptedException



##实现原理

CountDownLatch 通过重写 `tryAcquireShared`和`tryReleaseShared`来实现，前面说过 调用 await 方法以后，如果判断 state > 0，则会调用 AQS 里面的  `doAcquireSharedInterruptibly` 方法

<img src="https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/CountDownLatch%E8%AF%A6%E8%A7%A3/image-20211204230931758.png" alt="image-20211204230931758" style="zoom:50%;" />

这个方法首先会将当前线程包装成一个 node 并且初始化一个 FIFO 队列，如果只有一个线程调用 await 方法，那么通过 addWaiter 方法会形成一个只有两个节点的队列

> head -> curr（当前线程）

<img src="https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/CountDownLatch%E8%AF%A6%E8%A7%A3/image-20211204224850239.png" alt="image-20211204224850239" style="zoom:50%;" />

回到第一个图，当队列形成以后，就会调用 tryAcquireShared 来判断该线程是否允许执行

### tryAcquireShared

这个方法的含义就是在共享模式下，判断当前线程是否获取到执行的条件，注释上的英文简单翻译就是：这个方法每次都应该被调用，如果这个方法提示失败了，则当前线程就应该被入队然后等待其他线程唤醒该线程

而一旦满足了执行条件 `>0` 则会执行 setHeadAndPropagate 函数

<img src="https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/CountDownLatch%E8%AF%A6%E8%A7%A3/image-20211204231856852.png" alt="image-20211204231856852" style="zoom:50%;" />

这个函数也比较简单，首先就是将当前线程设置为 head 节点，然后判断是否需要唤醒后续节点，这里的 `propagate` 变量就是 tryAcquireShared 这个方法返回的，因此只要返回的是 > 0，那么就可以认为当前节点的任务已经结束了，可以唤醒后续的节点了

<img src="https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/CountDownLatch%E8%AF%A6%E8%A7%A3/image-20211204232243965.png" alt="image-20211204232243965" style="zoom:50%;" />

在这个方法里面，如果只有一个线程调用了 await 方法，那么第一个 其实 h == tail，而且 h == head，所以无需唤醒任何人，直接return 即可

而如果是多个线程都调用了 await 方法，那么此时该 node 后面还有 node，此时就需要唤醒后续节点了，而在 addWaiter 方法用的是 `Node(Thread thread, Node mode)`  构造函数，因此 ws 的值是 0 ，然后执行一个 CAS 操作，将 ws 的状态改成  PROPAGATE

<img src="https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/CountDownLatch%E8%AF%A6%E8%A7%A3/image-20211204233944643.png" alt="image-20211204233944643" style="zoom:50%;" />

上面几个状态值的大致意思如下：

* 1：已经取消
* 2：线程需要被唤醒
* 3：线程在条件队列里面
* 4：释放资源的时候需要通知其他共享节点

回到前面的一个图，如果 tryAcquireShared 返回的 <0，那么剩下的操作就是将该 node 的 ws 改成 -1，然后通过 `LockSupport.park(this)`挂起自己，最后等待被唤醒



## 疑问

在这里可能有人有疑问了，既然countDown方法是判断 `head !=null && head == tail` 来决定是否唤醒后续节点，那么假设在同一个时刻，countDown 将 state 改成 0 并且进入了如下的 if 判断，而另一个线程（线程2）正在入队操作，此时是否会出现另一个线程被挂起呢？

<img src="https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/CountDownLatch%E8%AF%A6%E8%A7%A3/image-20211204152730246.png" alt="image-20211204152730246" style="zoom:50%;" />

答案是不会的，首先 head 和 tail 节点都是 volatile，所以可以保证可见性，而且在前面提到过入队的时候，会通过 tryAcquireShared 判断当前的 state，所以一旦判断为 0 了，那么就会直接return，所以不会阻塞.



那么在 tryAcquireShared 判断之后，state 改成 0 了，这样其实也没啥问题，还是不会导致阻塞，在这个情况下，head 和 tail 肯定是有值的，如果 head == tail，那么此时 countDown 的线程直接break.

如果head 不等于 tail，那么就说明此时的队列情况如下：

> head ->  node

并且线程二最终执行的方法也是 doReleaseShared，而最终都是会通过 head == h 返回出去，所以也没有线程安全问题







