---
title: 线程的interrupt和stop区别，以及线程的中断机制
date: 2018-05-22 10:53:34
tags: [多线程]
categories: [Java,线程]
---
## interrupt
在Java里面线程的中断是一个协作式的，也就是说线程会在自己合适的时候自己中断自己，一般来讲线程如果需要中断的话有如下两种方法：
* 捕获InterruptException
* 通过Thread的`interrupted()`或者`isInterrupted()`方法，但是需要注意的是`interrupted`会清除这个线程的状态

当一个线程调用另一个线程的`interrupt`的时候，另一个线程并不会马上结束，而是会设置一个中断的状态，如果一个线程处于阻塞的状态，那么此时该线程会马上抛出一个InterruptException，由上层的代码进行处理。
若线程没有处于阻塞的话，此时线程还是会执行的。但是线程需要自己在合适的地方通过上述的两个方法来判断自己是否应该中断。如果自己

## stop
stop方法和interrupt有许多相似之处，具体就是在阻塞的时候都会抛出Interruptexception这个异常。
但是在运行期间的话`stop`方法会直接强迫另一个线程终止，并且抛出一个ThreadDeath的**Error**，这个


## 中断机制：
总结了下，有如下几种。
当线程在阻塞的时候收到中断请求，但是线程会抛出一个Exception并且线程的中断状态会被清除掉。（常见的阻塞如sleep，wait，join）等
> 此时需要注意，如果无法继续向上抛出这个异常的话，那么就应该继续调用`interrupt`这个方法，因为每一个中断都应该被上层知道的。

在判断线程的中断状态的时候需要注意
```java
while(!Thread.currentThread().isInterrupted()){
    try{
          // IO操作或者其他的中断操作
    }catch(InterruptException e){
        break;
    }
 
}
```
在这段代码中，IO操作一旦发生阻塞并且收到一个中断异常但是这个异常又没有进行处理，这个时候while会一直判断不到这个线程已经是需要被中断的了

## 总结
关于线程中断就是捕获的异常一定需要抛出，不能抛出就需要设置中断中断状态
