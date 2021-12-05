---
title: Error是真的不可以被捕获的吗?
date: 2018-05-25 16:06:59
tags: [Java]
categories: [Java,JDK]
---
在刚接触Java的时候经常听到的一句话便是在 Java 中，Exception 是可以捕获的，Error 是不可以捕获的。但是在随着学习的深入，会发现有些观点需要重新认识下了。
Throwable 这个类是自 JDK1.0 开始就存在于 Java 的语言之中。
## Throwable
首先引用一段 Oracle 官方文档上对 Throwable 的介绍[Java8 Thrwoable的介绍](https://docs.oracle.com/javase/8/docs/api/?java/lang/Error.html)：
> The Throwable class is the superclass of all errors and exceptions in the Java language. Only objects that are instances of this class (or one of its subclasses) are thrown by the Java Virtual Machine or can be thrown by the Java throw statement. Similarly, only this class or one of its subclasses can be the argument type in a catch clause. For the purposes of compile-time checking of exceptions, Throwable and any subclass of Throwable that is not also a subclass of either RuntimeException or Error are regarded as checked exceptions.
Instances of two subclasses, Error and Exception, are conventionally used to indicate that exceptional situations have occurred. Typically, these instances are freshly created in the context of the exceptional situation so as to include relevant information (such as stack trace data).
太长，省略大部分了......

简单翻译下就是，Throwable 是 Error 和 Exception 的父类，并且只能是 Error 和 Exception 的实例才可以通过 throw 语句或者 Java虚拟机 抛出异常。Exception 或者 Error 是在出错的情况下新创建的，从而将出错的信息和数据包含进去。
另外在这个文档中还提到了一点就是当低层方法向高层方法抛出异常的时候，如果抛出的异常是受检查的异常，则


## Error 
在看 Error 之前首先看一段代码，如下：
```java
public class ExceptionTest {
    private void test(){
        try{

        }catch (Error e){
            e.printStackTrace();
        }
    }
}
```
可以看见 Error 是可以被捕获的，虽然 Java 的catch语句可以捕获 Error，但是在Error的官方文档上却做了说明：不推荐对Error进行捕获，也就是说 Error 虽然可以被 Java 语言捕获，但是Java官方却是不推荐对Error进行捕获的。具体文档如下：
>  An Error is a subclass of Throwable that indicates serious problems that a reasonable application should not try to catch. Most such errors are abnormal conditions. The ThreadDeath error, though a "normal" condition, is also a subclass of Error because most applications should not try to catch it.
A method is not required to declare in its throws clause any subclasses of Error that might be thrown during the execution of the method but not caught, since these errors are abnormal conditions that should never occur. That is, Error and its subclasses are regarded as unchecked exceptions for the purposes of compile-time checking of exceptions.

也就是说 Error 的出现表示程序出现了严重的非正常问题，并且在Java总处于一些原因，Error 被认为是未检查的异常。


## 异常
异常分为受检查的异常和运行时异常，受检查的异常标志着程序在编译期间必须处理，常见的比如在读取一个文件的时候，Java语言必须要求抛出或者Catch FileNotFoundException。而运行时异常 Java 则是对其不做要求。

## 为什么Java不推荐捕获Error
Java官方文档的解释说是 Error 的出现代表的是一些严重的非正常的错误。那么在 Java 的官方文档中介绍的 Error 有如下几种：
> AnnotationFormatError, AssertionError, AWTError, CoderMalfunctionError, FactoryConfigurationError, FactoryConfigurationError, IOError, LinkageError, SchemaFactoryConfigurationError, ServiceConfigurationError, ThreadDeath, TransformerFactoryConfigurationError, VirtualMachineError。

这些Error的出现代表的是程序已经不用进行处理了，比如 OutOfMemoryError，如果出现了这个错误的话，那么程序已经无法运行下去了，此时捕获就没有意义了。

## 总结
所以并不是说Error是不可以捕获的，而是可以捕获的，但是 Java 官方并不推荐捕获Error。