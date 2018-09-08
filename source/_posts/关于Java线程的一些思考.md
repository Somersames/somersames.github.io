---
title: 关于Java线程的一些思考
date: 2018-02-07 17:20:50
tags: 多线程
categories: Java
---
# 消费者和生产者的模型：
今天在看《现代操作系统》P73页的一个关于多线程的竞态条件的时候书中说到了`唤醒等待位方法`，这个方法使我突然想起联想到以前在Java多线程中sleep()方法会清除中断状态的一些类似之处：
树上的代码：
```
#definde N 100
int count =0;
void producer(void){
int item;
while(True){
    item=producer_item();
    if (count == N) sleep();
    insert_item(item);
    count=count+1;
    if (count == 1 )wakeup(consumer);
}
}

void consumer(void){
    int item;
    while(true){
        if (count == 0)sleep();
        item =remove_item();
        count=count -1;
        if (count == N-1) wakeup(producer);
        consumer_item()
    }
}
```

在书中提到过一个情况就是：当消费者判断count==0的时候是会睡眠的，但是此时由于某种情况(线程的sleep()方法还没被执行完)，而恰巧生产者又生产了一个item，导致此时count加一为1，而当count为1的时候生产者是会发出一个wakeup信号给消费者的，此时，由于消费者并没有睡眠因此会忽略掉该信号，而当消费者真正睡眠之后又由于生产者再不会进行通知，导致队列被生产者塞满，从而该模型阻塞。
那么为了解决该方法，书中引入了`唤醒等待位方法`，该方法就是在生产者发出信号给消费者的时候添加一个中断状态，而当该线程需要进行睡眠的时候会先判断状态，若是wakeup则不进行睡眠

## Java多线程中的引用：
在Java中调用interrupt()方法的原理也是类似的，设置一个中断状态，但是这样做的原因是为了`安全起见`，因为通过其他的线程来中断另一个线程是及其不安全的，在其他线程发出中断信号的时候，它并不会知道另一个线程目前正在做什么事情，所以安全的做法是设置一个中断状态