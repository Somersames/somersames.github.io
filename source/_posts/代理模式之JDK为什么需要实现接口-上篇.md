---
title: 代理模式之JDK为什么需要实现接口(上篇)
date: 2020-04-15 22:58:59
tags: 动态代理
categories: Java
---
## 简介
首先代理模式分为静态代理和动态代理，由于JDK采用的是动态代理，所以静态代理在这里不再介绍。

## 动态代理
### JDK动态代理
首先`JDK`的动态代理要求真实的对象必须实现一个接口，而代理类则实现`InvocationHandler`接口来完成动态代理。如下代码：
> 接口



```java
public interface Car {
    void jdkProxy();
}
```
> 被代理的类
```java
public class Bus implements Car{
    @Override
    public void jdkProxy() {
        System.out.println("测试");
    }
}
```
> 代理类


```java
public class ProxyTest implements InvocationHandler {

    private Object target;

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("before---------");
        Object object = method.invoke(target,args);
        System.out.println("after---------");
        return object;
    }

    public Object getInstance(Object target) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        this.target = target;
        Class<?> clazz = target.getClass();
        return Proxy.newProxyInstance(new SelfClassLoader(),clazz.getInterfaces(),this);
    }
}
```
在代理模式中真实对象继承一个接口如果说是为了在代理类中统一管理，那么在 `JDK动态代理` 中如果被代理的对象还必须实现一个接口就有点繁琐了，但是对于 Java 语言来说其实也没办法。我们都认为很繁琐的一个实现，JDK 开发人员应该也会想到这一点。

下面就来解释下为什么动态代理必须实现一个接口。

#### 为什么必须实现一个接口
在回答这个问题之前，首先要说明的是 `JDK动态代理` 实现的原理是重新为对象类生成了一个子类，这个类由于是 JDK 自己生成的，于是会以`$`开头，在 JDK 文档中有如下说明：
```java
A proxy class has the following properties:

Proxy classes are public, final, and not abstract.
The unqualified name of a proxy class is unspecified. The space of class names that begin with the string "$Proxy" should be, however, reserved for proxy classes.
A proxy class extends java.lang.reflect.Proxy.
```
其实还有很多说明，但是涉及到本文的也就上面三条。
1. proxy类是非抽象类，且由`public final`修饰。
2. proxy类是以`$Proxy`开头的加类的名称。
3. proxy类是`java.lang.reflect.Proxy`的子类。

那么既然是`$Proxy`开头的，那么是否可以将生成的子类打印出来呢？，如下代码：


可以看到这个新生成的类其实是直接继承了 `Proxy` 类，由于Java的单继承方式，此时如果要获取对象类里面的方法那么就必须实现一个接口，所以也就对象类必须要实现一个接口。
```java
Car car = (Car) new ProxyTest().getInstance(new Bus());
```
那么当我们执行这个代码的时候究竟在执行什么呢？在这里 debug 来跟踪源码解释 JDK 到底是如何生产 Proxy 代理类的.
> newProxyInstance方法
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/newProxyInstance.png)

在这个方法里面，目前暂时需要注意红框中的方法，代理类是在该方法里面是生成的。

`Class<?> cl = getProxyClass0(loader, intfs);` 这一行代码，在这里返回的是一个 class，然后通过反射来初始化这个 class，最后这个 class 就是 JDK 生成的代理类。

> getProxyClass0


而在 `getProxyClass0` 方法里面就是通过 `proxyClassCache.get(loader, interfaces)`来获取 Proxy，如果里面包含该 classLoader 加载的 interface 的话，则直接返回，否则就是再初始化一次，然后返回。
```java
private static Class<?> getProxyClass0(ClassLoader loader,
                                           Class<?>... interfaces) {
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }

    // If the proxy class defined by the given loader implementing
    // the given interfaces exists, this will simply return the cached copy;
    // otherwise, it will create the proxy class via the ProxyClassFactory
    return proxyClassCache.get(loader, interfaces);
}
```

> proxyClassCache.get(loader, interfaces)


从这个方法之后就是真正开始生成 Proxy 类的过程，由于这里面包含一些 Java8 的 lambda表达式 用法，对于不太了解的人可能会有点看不懂。
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/proxy.png)

上面的图给出了初始化 map 的一个方法，在这里需要注意的是，这个map的结构如下：
```java
private final ConcurrentMap<Object, ConcurrentMap<Object, Supplier<V>>> map
        = new ConcurrentHashMap<>();
```
也就是一个两层的 ConcurrentMap，而之所以为为什么这样设计是跟实现的接口相关的，是由于第一层的 key 是由 classLoader 所生成的 key，而第二层测试keyX，与所实现的接口有关。
剩下的将会在下一篇文章。
