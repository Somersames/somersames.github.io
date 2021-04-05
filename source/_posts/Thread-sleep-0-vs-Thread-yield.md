---
title: Thread.sleep(0)-vs-Thread.yield()
date: 2020-07-07 00:25:44
tags: [java]
categories: Java
---
## 简介
`Thread.sleep(long)` 和 `yield()` 都表示的是让出当前线程的 `CPU` 时间片，两者在执行的时候，都不会去释放自己已经持有的锁。

### 多线程
在现代的处理器中，以 `4核心CPU` 为例，这表示在同一个时刻，只会有 4 个线程在并行执行，而在每一个核心内部，多个线程其实是顺序执行的，它们的执行顺序依赖于线程的调度算法。

### 线程调度算法
目前主要的调度算法有如下几种「可能不全」：
1. 先进先出（FIFO）
2. 最短耗时优先算法（SJF）
3. 时间片轮转算法（RR）
4. 优先级排序调度算法（PS）
5. 多级反馈队列算法（MLFQ）

以 `时间片轮转算法` 为例，多个线程每一个线程都会分到一定的执行时间，当本次执行时间结束以后，就会发生上下文切换。
同理，对于其他的调度算法，其实本质上都是一致的，就是在同一个时刻，一个核心只能执行一个线程，多线程其实就是通过CPU核心轮流执行线程。


在多线程编程中，如果需要暂停当前线程的执行，可以调用`Thread.sleep(long millis)` 方法，来让出`CPU` 时间片，即表示在接下来的 `millis` 毫秒内，该线程不再参与 `CPU` 时间片的竞争。

## 线程的状态
在 Java 中，可以通过执行 `jstack pid ` 命令来查看线程的运行状态，如下图所示：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/ThreadSleep/ThreadStatus_1.png)
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/ThreadSleep/ThreadStatus_2.png)

从上面两张图可以看到，目前在 Java 中，是已经含有四种状态了，那么还有两种没有列出来，它们分别是 `NEW`，`TERMINATED`
* `NEW`：这个状态表示的是线程刚刚创建，例如 `new Thread()`;
* `RUNNABLE`：其实这个状态，在 `Java` 中表示的是 `就绪` 和 `运行`
    *  `就绪` ：代表这个线程可以执行了，但是还在等待 `CPU` 的调度
    *  `运行` ：代表这个线程已经在执行中了
* `WAITING`：表示该线程正在等待一些条件，常见于调用了`Object.wait`,`Thread.join`,`LockSupport.park`
* `TIMED_WAITING`：对于线程的这种状态，常见于调用了 `Thread.sleep(long)` 或者 `Object.wait(long)` 方法等
* `BLOCKED`：常见于互斥锁的竞争
* `TERMINATED`：线程终止


## Sleep
那么当一个线程调用了 `Sleep` 方法之后，会由 `RUNNABLE` 转为 `TIMED_WAITING`，这个表示在接下来的 `long` 毫秒内，该线程不再参与 `CPU` 资源的竞争，需要注意的是 `sleep` 方法不会释放所持有的锁。

### Sleep(0)
那么当调用 `Sleep(0)` 的时候，究竟会发生什么，查看下 `Java` 的源代码，发现是一个 `native` 方法，于是查看 `hotspot` 方法，其代码如下：
```C
JVM_ENTRY(void, JVM_Sleep(JNIEnv* env, jclass threadClass, jlong millis))
  JVMWrapper("JVM_Sleep");

  if (millis < 0) {
    THROW_MSG(vmSymbols::java_lang_IllegalArgumentException(), "timeout value is negative");
  }

  if (Thread::is_interrupted (THREAD, true) && !HAS_PENDING_EXCEPTION) {
    THROW_MSG(vmSymbols::java_lang_InterruptedException(), "sleep interrupted");
  }

  // Save current thread state and restore it at the end of this block.
  // And set new thread state to SLEEPING.
  JavaThreadSleepState jtss(thread);

#ifndef USDT2
  HS_DTRACE_PROBE1(hotspot, thread__sleep__begin, millis);
#else /* USDT2 */
  HOTSPOT_THREAD_SLEEP_BEGIN(
                             millis);
#endif /* USDT2 */

  EventThreadSleep event;

  if (millis == 0) {
    // When ConvertSleepToYield is on, this matches the classic VM implementation of
    // JVM_Sleep. Critical for similar threading behaviour (Win32)
    // It appears that in certain GUI contexts, it may be beneficial to do a short sleep
    // for SOLARIS
    if (ConvertSleepToYield) {
      os::yield();
    } else {
      ThreadState old_state = thread->osthread()->get_state();
      thread->osthread()->set_state(SLEEPING);
      os::sleep(thread, MinSleepInterval, false);
      thread->osthread()->set_state(old_state);
    }
  } 
  // 忽略其他代码
#ifndef USDT2
  HS_DTRACE_PROBE1(hotspot, thread__sleep__end,0);
#else /* USDT2 */
  HOTSPOT_THREAD_SLEEP_END(
                           0);
#endif /* USDT2 */
JVM_END
```

从这段代码可以看到，在 `JVM` 中有一个选项，叫做 `ConvertSleepToYield`，这个参数默认是为 `true` 的，所以当调用 `Sleep(0)` 的时候，默认的会调用 `Thread.yield()` 方法。

那么如果关闭了这个选项的话，会调用 `os::sleep(thread, MinSleepInterval, false);` ,而 `MinSleepInterval` 的值是 `1`，所以如果关闭了这个选项的话，那么 JVM 首先会将当前的线程状态赋值为 `old_state`，然后通过 `os::sleep` 让其休眠 `1ms`，然后在将线程设置成原来的状态。


## yield
其实对于 `yield` 方法，jvm 也可以将其转换为 Sleep 方法：
```C
JVM_ENTRY(void, JVM_Yield(JNIEnv *env, jclass threadClass))
  JVMWrapper("JVM_Yield");
  if (os::dont_yield()) return;
#ifndef USDT2
  HS_DTRACE_PROBE0(hotspot, thread__yield);
#else /* USDT2 */
  HOTSPOT_THREAD_YIELD();
#endif /* USDT2 */
  // When ConvertYieldToSleep is off (default), this matches the classic VM use of yield.
  // Critical for similar threading behaviour
  if (ConvertYieldToSleep) {
    os::sleep(thread, MinSleepInterval, false);
  } else {
    os::yield();
  }
JVM_END
```

在 JVM 中，`ConvertYieldToSleep` 默认值是 `false`，所以如果不更改 JVM 的默认配置的话，`yield` 方法会调用 `os::yield();` 方法。


## 总结
其实对于 `Thread.sleep(0)` 和 `yield` 方法来讲，如果仅仅使用的是默认配置的话，那么它们最终调用的都是 `OS` 的 `yield` 方法。

那么其实对于它们来说，在执行的时候，都是让 `CPU` 再次发一个调度，如果当前的线程，没有被选中执行，那么它的状态就会由 `RUNNABLE` 变成 `WAITING`。