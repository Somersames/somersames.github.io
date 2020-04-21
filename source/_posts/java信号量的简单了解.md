---
title: java信号量的简单了解
date: 2019-12-04 00:20:45
tags:
---
<!-- 信号量在计算机的操作系统以及并发的时候经常使用到，用于多个线程之间的通信。（待定） -->

在Java语言里面，Semaphore 的作用是可以控制对于同一个临界资源，允许多少个线程同时执行。
当线程执行到临界区域的时候，需要先向 Semaphore 申请一个令牌，此时 Semaphore 会判断现有的的令牌是不是小于0，如果小于0，则阻塞当前线程，直至有线程将令牌归还回来。

## 申请到了permits
如果一个线程直接申请到了permits，则是直接通过CAS操作将 state 减一即可，然后线程继续执行。

## 无法申请到permits
当无法申请到 Semaphore 的 permits 的时候，则会将当前的线程进行阻塞，直到有线程执行完毕，释放了 permit。

### Semaphore 的类关系
那么在 `Semaphore` 里面，如果 `permits` 已经被降到0以下，按照信号量的规则，应当将当前线程阻塞，直到 `permits` 大于 0。查看 Semaphore 的类组织结构，会发现它的一些操作全部都是依赖于自己的一个 Sync 变量，此外还有两个变量用于实现公平锁和非公平锁，都是继承自 `Sync`，而 `Sync` 则是继承自 `AbstractQueuedSynchronizer` (下文简称AQS)。所以 Semaphore 的实现全部是依赖于 AQS。
以下为 Semaphore 的类图：
 ![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/Semphore%E7%9A%84%E9%9D%9E%E5%85%AC%E5%B9%B3%E4%BE%9D%E8%B5%96.png)


<!-- 在这里当获取的资源数已经成为了负数，那么首先会将上一个节点设置为`SINGLE`(-1)，这样的话，当上一个节点被释放或者被取消的话，那么它的后续节点就会运行， -->


<!-- AQS里面，独占锁的话是在释放锁的时候会进行通知下一个节点，但是共享锁却不是，它是在开始运行的时候就直接通知下一个节点， -->

下面是一个例子来说明 Semaphore 是如何工作的。

如下Demo
```java
public class SemaphoreTest implements Runnable {

    private static final Semaphore semaphore = new Semaphore(1);

    private int no;

    public static void main(String[] args) throws InterruptedException {
        for(int i =0 ;i< 2 ;i++){
            new Thread(new SemaphoreTest(i)).start();
        }
        Thread.currentThread().sleep(1000000);
    }

    public void run() {
        System.out.println(no + "start");
        try {
            semaphore.acquire();
        } catch (InterruptedException e) {  
            e.printStackTrace();
        }
        System.out.println(no);
        System.out.println("semaphore");
        try {
            Thread.currentThread().sleep(10000);
            semaphore.release();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public SemaphoreTest(int no) {
        this.no = no;
    }
}
```
在这个例子中，首先获取到 permits 的线程会执行，而第二个线程则会被阻塞，直到 permits 被释放。这也是 Semaphore 的目的，控制有多少个线程可以同时的访问某一个资源。


#### 设置permits

在 Semaphore 里面，直接使用 AQS 里面的`state`变量来决定允许多少个线程同时访问一个资源，通过 Semaphore 的构造函数就可以发现：
```java
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = -2694183684443567898L;

    NonfairSync(int permits) {
        super(permits);
    }

    protected int tryAcquireShared(int acquires) {
        return nonfairTryAcquireShared(acquires);
    }
}
```
该super就是调用 Sync 的构造函数，然后调用 AQS 里面的 `setState方法`，最后设置 AQS 的 state。

#### 获取permits
当一个线程尝试调用 acquire 方法的时候，最终会通过 NonfairSync 的 tryAcquireShared 方法，在该方法里面，会获取当前线程的 state，然后减去入参带过来的参数（默认是 1 ），最后判断是否小于 0，若大于 0 的话，则直接采用 CAS 的操作将剩余的 state 替换掉。小于 0 的话，直接返回并且会进入另一个方法。
```java
final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```
由于 state 是由 `volatile` 修饰的，所以说当一个线程原子性的修改了 state，另一个线程也会立即获取到 state 的最新状态的。
这个方法不是一个原子性的，但是由于 CAS 操作是原子性的，那么最终还是能保证一致性的。如下：
1. 如果一个线程在获取到了 available 恰好为 1， 准备减去 1 的时候，此时另一个线程恰好 CAS 执行完毕，将 state 更新为 0 了，此时第一个线程在执行 CAS 的时候，由于 expect 已经修改为 0 了，所以此时 CAS 操作一定不成功。
2. 如果在 getState 之前，state 已经被修改成 0 的，那么由于 `remaining < 0`，所以直接返回 remaining。

#### 线程阻塞
在这里先简单介绍下线程的阻塞方式：
首先生成一个 Node，然后再判断下队列的尾部 tail 是不是为 null，如果是的话初始化 head 和 tail，并且将当前线程的 Node 的前置节点指向 head，然后通过 CAS 操作将 tail 设置为当前线程创建的 node， 最后将 head 的后置节点指向该节点。

然后再判断前置节点是不是 head，是的话就再次尝试获取 state，若获取不到则直接将前一个节点的 waitStatus 修改为 -1（即后置节点的线程需要被唤醒）然后直接通过`LockSupport.park(this)`将其阻塞。

当 CAS 操作失败，一般都是 state 已经小于 0 了，此时就会进入 doAcquireSharedInterruptibly 方法里面，在该方法里面会使用 AQS 里面的 FIFO 队列，来实现对于临界资源的控制。

##### 详细
Semaphore 首先会进行创建一个 Node，在首次阻塞某一个线程的时候，由于 tail 为null，所以会直接进入 enq 方法，在该方法里面会将 tail 和 head 都设置为一个新的 node，然后展开第二次的循环，最后在将当前线程的 node 的 prev 指向 head，通过CAS操作将 tail 指定为该node，最后将 head 的 next 指向 该node（注意这一步为非原子性的，所以会影响到后面的release方法的一个判断）
```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

到这里，一个 FIFO 队列就生成了，此时 head 为一个node（waitStatus为0，next为被阻塞的线程），当从 addWaiter 方法返回的时候，Semaphore 还会进行一个判断，如果当前线程的前置节点是 head，就再次尝试一次获取 state，如果获取不到的话就将前置节点的 waitStatus 设置成 -1，然后通过`LockSupport.park(this)`将本线程中断，至此该线程被阻塞了。

**如果在判断前置节点是 head 之后，然后通过 tryAcquireShared 获取到了 state** 那么此时就会调用 setHeadAndPropagate 将自己设置为头节点，同时判断是否需要唤醒后续的节点



## 释放
假如一个线程执行完毕之后，是会调用 release 方法来释放资源的，首先还是以 CAS 操作原子性的增加 state，
```java
protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        int next = current + releases;
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
        if (compareAndSetState(current, next))
            return true;
    }
}

private void doReleaseShared() {
    /*
        * Ensure that a release propagates, even if there are other
        * in-progress acquires/releases.  This proceeds in the usual
        * way of trying to unparkSuccessor of head if it needs
        * signal. But if it does not, status is set to PROPAGATE to
        * ensure that upon release, propagation continues.
        * Additionally, we must loop in case a new node is added
        * while we are doing this. Also, unlike other uses of
        * unparkSuccessor, we need to know if CAS to reset status
        * fails, if so rechecking.
        */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                        !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}

private void unparkSuccessor(Node node) {
    /*
        * If status is negative (i.e., possibly needing signal) try
        * to clear in anticipation of signalling.  It is OK if this
        * fails or if status is changed by waiting thread.
        */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
        * Thread to unpark is held in successor, which is normally
        * just the next node.  But if cancelled or apparently null,
        * traverse backwards from tail to find the actual
        * non-cancelled successor.
        */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```
当 CAS 操作成功之后返回 true，然后调用 doReleaseShared 方法，在该方法里面，会首先将 head 的状态更改为 0，然后然后通过 unparkSuccessor 来唤醒后面的线程，在这里需要注意的是那一个 for 循环里面的代码。
### 为什么for循环的时候需要从尾节点开始
在这里其实是由于前面在设置尾节点的时候 CAS 虽然是一个原子性操作，但是在 CAS 操作之后，紧跟着 `pred.next = node` 这一步为非原子性的，所以就导致了有可能从头遍历的时候会断开掉，所以此时就需要从尾节点开始，因为这样一定不会断开的，也是最有保证的。