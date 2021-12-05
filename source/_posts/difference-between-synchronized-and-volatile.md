---
title: 浅谈 synchronized 和 volatile
date: 2021-05-30 00:30:34
tags: [并发]
categories: [Java,JDK]
---
# 一、锁
从细节上来说，锁分为 乐观锁、悲观锁

乐观锁适用于读多写少的场景，一般是通过 CAS 进行操作，因为不用加锁，所以性能上比悲观锁优秀太多

悲观锁适用于写多读少的场景，性能开销比较大。

## 1：乐观锁
在 Java 中的 `Unsafe` 的 `compareAndSwapInt` 就是用到了这个特性

但是乐观锁还会产生一个 `ABA` 的问题，一般是通过 `version版本号` 的方式来解决
另外一个就是竞争问题，如果大量的线程都在进行 CAS 操作，那么势必会造成某些线程 CAS 操作非常耗时，会白白的浪费非常多的 CPU 资源，甚至造成 CPU 100% 的情况


## 2：自旋锁
当一个线程获取不到锁的时候会进行阻塞，而阻塞一个线程是需要进行上下文切换，这些都是需要 CPU 进行操作，加入切换一个线程的上下文所需要的时间是 10，而代码的同步快执行一次是 5，那么此时进行线程的上下文切换是不值得

而现在的 CPU 一般都是多核心处理器，为了提高鲜绿，自选锁也就出现了。两个线程并行执行，让后面那个线程不进入阻塞状态，而是自旋等待 CPU 获取锁。

在 Java 中的 `AtomicInteger` 中也是用到了这个特性
```java
public final int updateAndGet(IntUnaryOperator updateFunction) {
    int prev, next;
    do {
        prev = get();
        next = updateFunction.applyAsInt(prev);
    } while (!compareAndSet(prev, next));
    return next;
}
```
会一直进行 cas 操作，直至设置成功


### 2.1：缺点
自选锁的缺点就是需要依赖临界资源的执行时间，如果临界资源执行的时间太长，会导致 CPU 时间被浪费


## 3：适应性自选锁
为了弥补自选锁的缺点，在 JDK1.6 里面引入了适应性自选锁，线程的自选状态会根据锁获取的状态以及上一次自选获取锁的时间来决定此次的自选次数


# 二、synchronzed
synchronzed 是 Java 中的一个关键字，由 JVM 负责实现，可以加在代码块、实例方法、静态方法上，加在不同的地方，锁住的对象是不同的。
代码块：锁住的是 () 内的对象
实例方法：锁住的是当前对象实例
静态方法：锁住的是当前对象的Class

synchronzed 具有原子性、互斥性、可重入性、不可中断性


## 1：原子性
首先来解释下原子性的定义：一个操作要么全部执行完成，不执行。
那么 synchronized 是如何保证的呢？
synchronized 修饰的代码块或者方法，在同一个时刻，只有一个线程可以获得锁并执行，其他线程只能等锁释放以后才可以执行，否则会阻塞。
这样就保证了在同一个时刻，只有一个线程可以对变量进行修改，保证了原子性

## 2：可重入性
对于同一个线程加的锁，该线程可以在任何时间再次执行同步代码块内部的方法

## 3：实现细节

如果对实例方法或者静态方法加锁，在编译后的 class 文件会出现两个 flag，代表这个方法是一个同步方法，其他线程在执行这个方法的时候必须获得锁
如果修饰的是实例方法或者静态方法，那么在反编译的字节码里面可以看到如下这个关键字：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/difference%20between%20synchronized%20and%20volatile/ACC_SYNCHRONIZED%402x.png)

如果修饰的是代码块，那么编译成字节码后的代码如下：

![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/difference%20between%20synchronized%20and%20volatile/monitor_enter_exit.png)

### 3.1：MarkWord
在谈及 synchronized 的时候，先了解下 MarkWord，Java 的对象头由三部分组成：Mark World、对象引用指针、数组长度（数组才有）

而 synchronized 的一些操作主要就在 MarkWord 中

下面是来自于 markOop.hpp 中关于 MarkWord 区域的描述
```java
//  32 bits:
//  --------
//  hash:25 ------------>| age:4    biased_lock:1 lock:2 (normal object)
//  JavaThread*:23 epoch:2 age:4    biased_lock:1 lock:2 (biased object)
//  size:32 ------------------------------------------>| (CMS free block)
//  PromotedObject*:29 ---------->| promo_bits:3 ----->| (CMS promoted object)
//
//  64 bits:
//  --------
//  unused:25 hash:31 -->| unused:1   age:4    biased_lock:1 lock:2 (normal object)
//  JavaThread*:54 epoch:2 unused:1   age:4    biased_lock:1 lock:2 (biased object)
//  PromotedObject*:61 --------------------->| promo_bits:3 ----->| (CMS promoted object)
//  size:64 ----------------------------------------------------->| (CMS free block)
```
如果画成图则是如下形式
![32位机器](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/difference%20between%20synchronized%20and%20volatile/32.png)
![64位机器](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/difference%20between%20synchronized%20and%20volatile/64.png)
### 3.2：加锁过程
synchronized 在 JDK1.6 之后的到了明显的优化，由之前的直接到重量级锁变为如下几个步骤：

无锁、偏向锁、轻量级锁、重量级锁


### 3.3无锁
这是最理想的情况，当处于无锁的状态下时，Mark World 存放的是对象的 hashCode，以及当前对象的分代年龄。这里所说的 HashCode 指的是一致性 HashCode，也就是通过 Object::hashCode 或者 System::identityHashCode(Object) 方法 得到的

> 关于一致性 HashCode：


因为一致性 HashCode 是一个随机数，第二次计算肯定与第一次计算得到不同的结果，而在 JVM 中一个对象的 HashCode 前后调用需要保持一致，所以这也是为什么一个对象生成过一致性 HashCode 以后便无法再次进入偏向锁状态

而如果一个对象正处于偏向锁状态，但是立即收到了重新计算一致性 HashCode 的请求，那么此时就会马上被膨胀为重量级锁


### 3.4偏向锁：
引入偏向锁的目的是为了解决多线程竞争不激烈的情况下，例如一个程序在大多数情况下只有一个线程进行一次或者多次的访问，对于这种情况，就没必要进行加锁的操作了。在 JDK6 之前，synchronized 之所以性能很差和每次加锁都是重量级锁有关

持有偏向锁的线程不会主动的撤销自己所持有的偏向锁，如果此时发生了竞争，那么当前的业务线程需要向 VmThread 请求进入到全局的安全点，一旦进入到安全点，就会尝试撤销偏向锁，如果此时发现之前持有偏向锁的线程已经退出同步代码块或者已经结束，则直接进行 CAS 替换 MarkWord 的 ThreadId。

否则，此时会将偏向锁撤销，并且设置锁标记为轻量级锁，持有偏向锁的线程不会主动的撤销偏向锁
```c
void ObjectSynchronizer::fast_enter(Handle obj, BasicLock* lock, bool attempt_rebias, TRAPS) {
 if (UseBiasedLocking) {
    if (!SafepointSynchronize::is_at_safepoint()) {
      BiasedLocking::Condition cond = BiasedLocking::revoke_and_rebias(obj, attempt_rebias, THREAD);
      // 如果是已经撤销并且重新偏向成功，直接返回
      if (cond == BiasedLocking::BIAS_REVOKED_AND_REBIASED) {
        return;
      }
    } else {
      // 如果已经在安全点，已经发生竞争，直接尝试撤销并且升级
      assert(!attempt_rebias, "can not rebias toward VM thread");
      BiasedLocking::revoke_at_safepoint(obj);
    }
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
 }
 // 升级轻量级锁
 slow_enter (obj, lock, THREAD) ;
}
```

### 3.5轻量级锁：
如果一个锁需要升级为轻量级锁，会进行如下操作：

首先检查是否是无锁状态，如果是的话，则在当前的线程栈桢里面新建一个 LockRecord 的记录，官方命名为 displaced mark，
然后将对象的 MarkWord 拷贝一份进 displaced mark，再通过 CAS 操作将对象中的 MarkWord 指针更新为 LockRecord 的地址，并且将 LockRecord 中的 owner 更新为原 MarkWord

如果是已经有锁的状态下，会检查对象的 MarkWord 指针是否指向自己的一个栈桢，如果是的话，则代表是自己重入，那么直接执行即可

否则会进行自旋，如果达到一定的阈值后，还无法获取到锁，则直接升级为重量级锁

```c
void ObjectSynchronizer::slow_enter(Handle obj, BasicLock* lock, TRAPS) {
  markOop mark = obj->mark();
  // 必须是非偏向锁状态
  assert(!mark->has_bias_pattern(), "should not see bias pattern here");

  // 判断是否有锁，如果 JVM 没有开启偏向锁，那么此时可能会一种无锁的状态
  if (mark->is_neutral()) {
    // Anticipate successful CAS -- the ST of the displaced mark must
    // be visible <= the ST performed by the CAS.
    // 设置 displaced MarkWord
    lock->set_displaced_header(mark);
    if (mark == (markOop) Atomic::cmpxchg_ptr(lock, obj()->mark_addr(), mark)) {
      TEVENT (slow_enter: release stacklock) ;
      return ;
    }
    // Fall through to inflate() ...
  } else
  if (mark->has_locker() && THREAD->is_lock_owned((address)mark->locker())) {
    assert(lock != mark->locker(), "must not re-lock the same lock");
    assert(lock != (BasicLock*)obj->mark(), "don't relock with same BasicLock");
    lock->set_displaced_header(NULL);
    return;
  }

#if 0
  // The following optimization isn't particularly useful.
  if (mark->has_monitor() && mark->monitor()->is_entered(THREAD)) {
    lock->set_displaced_header (NULL) ;
    return ;
  }
#endif

  // The object header will never be displaced to this lock,
  // so it does not matter what the value is, except that it
  // must be non-zero to avoid looking like a re-entrant lock,
  // and must not look locked either.
  lock->set_displaced_header(markOopDesc::unused_mark());
  ObjectSynchronizer::inflate(THREAD, obj())->enter(THREAD);
}
```
升级为轻量级锁以后，竞争锁的线程会自旋几次，避免升级至重量级锁，这个自旋次数不是一个固定值，而是由 JVM 动态来决定的

如果自旋次数达到阈值，那么会直接升级为重量级锁，或者 JVM 判断之前几次自旋都没有获取到锁，那么也就不用再自选了，因此可能会直接升级到重量级锁


### 3.6重量级锁
重量级锁是通过 mutex 来实现的，锁的状态会被改成「10」，并且 MarkWord 里面存储的是指向重量级锁的指针，所有等待的线程都会被挂起


# 三、volatile
volatile 和 synchronized 不同的是，volatile 只有可见性，没有互斥性

## 1可见性
在计算机发展的早前，CPU 多是单核的，虽然仍然可以并发的执行程序，但是不会存在可见性问题，因为所有的数据要么存在于内存中，要么存在于 CPU 的缓存中，所以任何线程在数据写入和读取的时候一定都是从同一个地方获取的。

随着计算机的发展，多核心的 CPU 出现了，这个时候程序执行的速度可以加快，但是前辈们发现计算机从内存中获取数据的速度依然很慢，因此考虑要不要直接在 CPU 内部做一个缓存，将内存中的数据和指令批量读取一部分来到 CPU 本地来，然后进行处理，最后将处理完数据再回写到内存中

但是这样就会产生一个可见性问题：假设有两个 CPU 同时将一个变量 a++  读取到了自己的 CPU 缓存中，同时执行了指令 +1，于是在回写的时候，都将 a 的值写为 1 了，但是程序正常执行的话，a 的值应该是 2，而不是1

### 1.1总线加锁
> 总线是与所有 CPU 相连接的一个主线路，可以理解为所有的 CPU 指令都必须经过总线
当一个 CPU 读取变量 a 到自己本地的时候，会向总线发送一个 LOCK 信号，此时其他 CPU 便会暂停执行，直至第一个 CPU 执行完毕


### 1.2MESI 协议
MESI 是一种基于实效的缓存一致性协议，通俗来说就是当一个 CPU 修改了一个变量的值以后，其他的 CPU 会立马感知到并且将自己的本次缓存设置为无效，而 MESI 对应的四个状态分别是
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/difference%20between%20synchronized%20and%20volatile/mesi.png)

每一次 CPU 从内存中读取数据的时候，都会向其他 CPU 发送一个事件，其他 CPU 接收到该事件以后，都会给到相应。
关于向消息总线发送的消息，感兴趣的话可以去看下[维基百科](https://zh.wikipedia.org/wiki/MESI%E5%8D%8F%E8%AE%AE)


## 2禁止重排序
volatile 关键字还有一个作用就是禁止重排序，该实现该功能依赖于内存屏障，在 hotspot 中，内存屏障如下有如下几个：
LoadLoad、LoadStore、StoreLoad、StoreStore

而在 openJdk 里面，对这几个屏障有很详细的描述[openJdk关于这几个的描述](https://github.com/openjdk/jdk/blob/6bab0f539fba8fb441697846347597b4a0ade428/src/jdk.internal.vm.ci/share/classes/jdk.vm.ci.code/src/jdk/vm/ci/code/MemoryBarriers.java)

在 Java 中，对于这几个命令可以简化为如下：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/difference%20between%20synchronized%20and%20volatile/memory_barrier.png)
该图表示的是在第一个操作之后，JVM 需要在第二个操作之前加入的一个内存屏障，正是这些操作才会使得 volatile 可以拥有禁止重排序的功能
