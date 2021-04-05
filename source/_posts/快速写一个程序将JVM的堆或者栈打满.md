---
title: 快速写一个程序将JVM的堆或者栈打满
date: 2020-07-11 16:03:11
tags: [JVM]
categories: JVM
---

# 堆和栈
## 数据结构
首先「堆」或者「栈」在本质上其实是一个数据结构，简介如下：
「栈」既可以用链表来实现，又可以用数组来实现，用链表来实现的话，它是一个含有头指针和尾指针的一种数据结构，根据含有指针的不同，分为单链表和双向链表。
「堆」是一种类似于「完全二叉树」的数据结构


## JVM中的堆和栈
在 JVM 中由于编译后的 class 文件都是一行行的指令，因此天然适合用「栈」这种数据结构，
```java
public class JvmTest {
    public static void main(String[] args) {
        int i = 0;
        i += 1;
        System.out.println(i);
    }
}

```

反编译后如下：
```C
Compiled from "JvmTest.java"
public class JvmTest {
  public JvmTest();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: iconst_0
       1: istore_1
       2: iinc          1, 1
       5: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       8: iload_1
       9: invokevirtual #3                  // Method java/io/PrintStream.println:(I)V
      12: return
}
```


那么对于这种指令来讲，以一种先进先出的方式存入肯定是最好的，所以在 JVM 中，栈的组成是一个一个的栈帧，那么每一个栈帧都包含如下几个部分：
```java
局部变量表
操作数栈
动态链接
方法出口
```
以 `HotSpot VM` 为例，由于其采用了固定栈大小的实现，也就指定了每一个线程的所能分配的栈内存的大小，那么依据此思路，可以有如下两种方法造成栈溢出：
1. 无限递归，导致无法为该线程分配内存
2. 无限创建线程，导致无法为该线程分配足够的内存

> `-Xss` 可以指定栈的大小


### demo1:
注意加上启动参数：`-Xss16m`
```java
public class StackOverFlowTest {
    public static void main(String[] args) {
        new StackOverFlowTest().over(1);
    }
    private void over(int deepth){
        System.out.println(deepth);
        over(++deepth);
    }
}
```

不一会就可以看到 `java.lang.StackOverflowError`

### demo2:
```java
public class StackOverFlowTest {

    public static void main(String[] args) {
        new StackOverFlowTest().threadOver();
    }
    private void threadOver(){
        ThreadTest threadTest = new ThreadTest(1);
        while (true){
            new Thread(threadTest).start();
        }
    }
    class ThreadTest implements Runnable{

        public ThreadTest(int depth) {
            this.depth = depth;
        }

        private int depth;
        @Override
        public void run() {
            System.out.println("thread start" + depth);
            try {
                Thread.sleep(10000000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```
艹，运行到一半的时候，把 mac 搞挂了。
正常情况下是会出现
```java
java.lang.OutOfMemoryError: Unable to create new native thread
```


## 打满堆：
在 `JVM8` 中，堆一般存放的是 `对象实例`，含有 `S0`、`S1`、`Eden`、`Old`这几个区域，具体的可以查看 JVM 规范。
```java
// -Xmx32m
public class JvmTest {
    public static void main(String[] args){
        List<VM> list = new ArrayList<>();
        while (true){
            VM vm = new VM();
            list.add(vm);
        }
    }
    static class VM{
        private byte[][] b = new byte[1024][1024];

    }
}
```
运行一段时间就会出现：
```java
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at jvm.over_test.JvmTest$VM.<init>(JvmTest.java:30)
	at jvm.over_test.JvmTest.main(JvmTest.java:24)
```
## 元数据区：
首先 `元数据区` 保存的是 **类信息、运行时常量池、以及即时编译器编译后的代码** 等。那么根据存储的数据的特性，直接选择动态生成代理类的加载类信息来打满这个区域，需要注意的是这个区域使用的是机器的内存，所以在测试的时候最好指定下 `元数据区` 的大小。
> 在这里也可以用自定的 `ClassLoader`，但是自定义 `ClassLoader` 需要重写 `loadClass` 方法，比较麻烦，所以直接选择使用代理类
```java
// -XX:MaxMetaspaceSize=16M -XX:MetaspaceSize=16M 
public class JvmTest {
    public static void main(String[] args) throws MalformedURLException{
        List<InterfaceA> list = new ArrayList<>();
        String classFilePath = "file:/Users/sunzhaohui/Desktop/person/java/MyCsNote/jvm/src/main/java/jvm/StringTest";
        URL[] classFileUrl = new URL[]{new URL(classFilePath)};
        URLClassLoader newClassLoader = new URLClassLoader(classFileUrl);
        while (true) {
            InterfaceA t = (InterfaceA) Proxy.newProxyInstance(newClassLoader, new Class<?>[]{InterfaceA.class}, new MyInvocationHandler(new InterfaceAImpl()));
            list.add(t);
        }
    }
}
```