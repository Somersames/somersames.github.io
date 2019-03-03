---
title: java异步IO的回调机制
date: 2019-01-24 00:05:48
tags: [java]
categories: Java
---
## 异步
在Java的nio里面，经常遇到的一个词语是`回调`，一个主线程负责分发各种请求至子线程，同时子线程处理完毕之后通知主线程，这其中就涉及到了回调机制。

在Java中，异步IO的回调方式主要是`CallBack`和`Future`。

由于`Future`获取结果是一种阻塞的方式，所以本次就主要来了解`Callback`回调方式的运行机制。

由于在异步IO里面，主线程不需要等待子线程来获取结果，所以可以极大的提高程序运行的效率，但是子线程必须在完成之后通知父线程，于时这就引出了回调。

在Java中回调是通过一个匿名对象来实现，每一个线程子线程在运行的时候都会传入一个匿名的对象，然后子线程完成任务之后，通过调用该对匿名对象来进行回调

### 代码：
#### 回调接口
```java
public interface ComplateHande {
    void complated();
    void fial();
}

```

#### 子线程
```java
public class ClientServer extends Thread{

    private ComplateHande complateHande;

    private int sleepTime;
    private String message;
    public ClientServer(ComplateHande complateHande,String message,int sleepTime) {
        this.message= message;
        this.sleepTime = sleepTime;
        this.complateHande = complateHande;
    }

    public ClientServer() {
    }

    @Override
    public void run() {
        try {
            comulate();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    private void comulate() throws InterruptedException {
        Thread.currentThread().sleep(sleepTime*1000);
        System.out.println(this.message);
        this.complateHande.complated();
    }

}

```

#### Main方法
```java
public class MainServer {
    public static void main(String[] args) throws InterruptedException {
        ClientServer clientServer1 =new ClientServer(new ComplateHande() {
            @Override
            public void complated() {
                System.out.println("处理完毕");
            }

            @Override
            public void fial() {
                System.out.println("处理失败");
            }
        },"一号任务",2);
        clientServer1.start();
        ClientServer clientServer2 =new ClientServer(new ComplateHande() {
            @Override
            public void complated() {
                System.out.println("2处理完毕");
            }

            @Override
            public void fial() {
                System.out.println("处理失败");
            }
        },"二号任务",1);
        clientServer2.start();
        ClientServer clientServer3 =new ClientServer(new ComplateHande() {
            @Override
            public void complated() {
                System.out.println("3处理完毕");
            }

            @Override
            public void fial() {
                System.out.println("2处理失败");
            }
        },"三号任务",1);
        clientServer3.start();
        ClientServer clientServer4 =new ClientServer(new ComplateHande() {
            @Override
            public void complated() {
                System.out.println("4处理完毕");
            }

            @Override
            public void fial() {
                System.out.println("处理失败");
            }
        },"四号任务",3);
        clientServer4.start();
        Thread.currentThread().sleep(1000);
        System.out.println("自己处理自己的事情+1");
        Thread.currentThread().sleep(1000);
        System.out.println("自己处理自己的事情+1");
        Thread.currentThread().sleep(1000);
        System.out.println("自己处理自己的事情+1");
        Thread.currentThread().sleep(1000);
        System.out.println("自己处理自己的事情+1");
        Thread.currentThread().sleep(10000);
    }
}

```
#### 运行结果
```java
二号任务
2处理完毕
三号任务
3处理完毕
自己处理自己的事情+1
一号任务
自己处理自己的事情+1
处理完毕
四号任务
4处理完毕
自己处理自己的事情+1
自己处理自己的事情+1
```

## 其他方式
在`nio`里面，其实还有其他的实现方式，比如通过建立一个主线程的阻塞队列，然后分发任务至子线程，子线程若处理完毕，则提交任务至主线程