---
title: 代理模式之JDK为什么需要实现接口(下篇)
date: 2020-04-21 01:00:44
tags: [Java]
categories: Java
---

# 前言
在上篇主要介绍了 JDK 动态代理执行的一些流程，通过 Proxy 类来实现生成新的代理类，在这一篇主要讲的是 Proxy 是如何来生成以及缓存生成的代理类

## 二级缓存
在上一篇的结尾提到过了一个map，其结构如下：
> ConcurrentMap<Object, ConcurrentMap<Object, Supplier<V>>>

在这里可以看到是两个 `ConcurrentMap` 来实现的嵌套，那么 Java 为什么要这样设计呢？首先查看 `CacheKey` 的代码。如下：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/JDK%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86/CacheKey%E7%9A%84%E7%94%9F%E6%88%90.png)

可以看到 `CacheKey` 是继承自 WeakReference，以便于当该 `CacheKey` 没人使用的时候可以让 GC 直接回收。`ReferenceQueue` 是 WeakReference 所使用的。

那么对于 Java 来讲，其是用一个 classLoader 为 key 的 cacheKey 来生成一个 key，这个key就是二级缓存的第一个key。回到上述的map。
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/JDK%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86/map%E7%9A%84%E4%BA%8C%E7%BA%A7%E7%BC%93%E5%AD%98.png)

可以看到在首次初始化的时候，map 还是会以刚刚的 `cacheKey` 来获取二级缓存，但是首次一定是空的，所以此时直接 new 了一个 `ConcurrentMap` 作为 `value` 来放入二级缓存。
再往下看代码：
```java
Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));
```
这是一个 Java8 的 lambda 表达式，查看这个表达式实现的地方：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/JDK%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86/keyFactory.png)

![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/JDK%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86/KeyX%E6%9E%84%E9%80%A0%E5%87%BD%E6%95%B0.png)

上面的 `KeyX` 的作用是用被代理的人所实现的接口进行hash，同时将它们实现的接口放入 `refs`，这个`refs`是一个数组，
> 这个 refs 和 WeakReference 有关，主要是用于当 WeakReference 的对象被回收的时候作为一个监听

此时这个 subKey 已经生成了，那么接下来就是
>  Supplier<V> supplier = valuesMap.get(subKey);

这个也是 Java8 的 lambda 函数，具体作用是通过调用其 get 方法，可以快读的执行函数，但是这里，这个 `supplier` 是由 `Factory` 所实现的。
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/JDK%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86/supplier.png)

这一段代码的具体逻辑是，当 `supplier` 不为空的时候就直接通过 `get` 方法获取其 value，然后返回，但是如果为 `null` 的话，那么首先就要初始化一个 `Factory`，然后在这期间，如果有现成将 `supplier` 赋值了，那么此时就会进行空判断，如果替换 `valueMap` 成功，则直接返回刚刚生成的 `factory`，否则返回 `valueMap` 里面的值。

在这里需要注意的是这个 value 就是一个代理类的的类名称。
## 代理类生成
```java
@Override
public synchronized V get() { // serialize access
    // re-check
    Supplier<V> supplier = valuesMap.get(subKey);
    if (supplier != this) {
        // something changed while we were waiting:
        // might be that we were replaced by a CacheValue
        // or were removed because of failure ->
        // return null to signal WeakCache.get() to retry
        // the loop
        return null;
    }
    // else still us (supplier == this)

    // create new value
    V value = null;
    try {
        value = Objects.requireNonNull(valueFactory.apply(key, parameter));
    } finally {
        if (value == null) { // remove us on failure
            valuesMap.remove(subKey, this);
        }
    }
    // the only path to reach here is with non-null value
    assert value != null;

    // wrap value with CacheValue (WeakReference)
    CacheValue<V> cacheValue = new CacheValue<>(value);

    // put into reverseMap
    reverseMap.put(cacheValue, Boolean.TRUE);

    // try replacing us with CacheValue (this should always succeed)
    if (!valuesMap.replace(subKey, this, cacheValue)) {
        throw new AssertionError("Should not reach here");
    }

    // successfully replaced us with new CacheValue -> return the value
    // wrapped by it
    return value;
}
```
这里还有一个重要的方法就是 `Factory.get()`，该方法由 `synchronized` 修饰，是用于生成代理类的 class 文件的。而生成代理类的方法在 `ProxyClassFactory` 的 apply 里面。
```java
@Override
public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {
    Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
    for (Class<?> intf : interfaces) {
       // ...... 省略
        if (!interfaceClass.isInterface()) {
            throw new IllegalArgumentException(
                interfaceClass.getName() + " is not an interface");
        }
    }
    // ...... 省略
    for (Class<?> intf : interfaces) {
        int flags = intf.getModifiers();
        if (!Modifier.isPublic(flags)) {
            accessFlags = Modifier.FINAL;
            String name = intf.getName();
            int n = name.lastIndexOf('.');
            String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
            if (proxyPkg == null) {
                proxyPkg = pkg;
            } else if (!pkg.equals(proxyPkg)) {
                throw new IllegalArgumentException(
                    "non-public interfaces from different packages");
            }
        }
    }
    //...... 省略
    /*
     * Choose a name for the proxy class to generate.
     */
    long num = nextUniqueNumber.getAndIncrement();
    String proxyName = proxyPkg + proxyClassNamePrefix + num;

    /*
     * Generate the specified proxy class.
     */
    byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
        proxyName, interfaces, accessFlags);
    try {
        return defineClass0(loader, proxyName,
                            proxyClassFile, 0, proxyClassFile.length);
    } catch (ClassFormatError e) {
        /*
         * A ClassFormatError here means that (barring bugs in the
         * proxy class generation code) there was some other
         * invalid aspect of the arguments supplied to the proxy
         * class creation (such as virtual machine limitations
         * exceeded).
         */
        throw new IllegalArgumentException(e.toString());
    }
}
}
```
在这里有几个重要的方法说下：
```java
// 判断是否是接口，这也是为什么 JDK动态代理必须实现一个接口了，因为不是接口的话这个地方验证过不去
if (!interfaceClass.isInterface()) {
    throw new IllegalArgumentException(
        interfaceClass.getName() + " is not an interface");
}

```
至于生成的类，可以通过生成文件来查看了。当生成完 class 文件之后，回到 `Proxy.newProxyInstance` 方法。

```java
final Constructor<?> cons = cl.getConstructor(constructorParams);
final InvocationHandler ih = h;
if (!Modifier.isPublic(cl.getModifiers())) {
    AccessController.doPrivileged(new PrivilegedAction<Void>() {
        public Void run() {
            cons.setAccessible(true);
            return null;
        }
    });
}
return cons.newInstance(new Object[]{h});
```
在这里是将参数上的 `InvocationHandler` 通过构造器将代理类中的 `InvocationHandler` 赋值，从而使用
`invoke` 方法来实现代理。

## 生成的代理类
```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

import design_model.proxy.jdk.Car;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class $Proxy0 extends Proxy implements Car {
    private static Method m1;
    private static Method m3;
    private static Method m2;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final void run() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m3 = Class.forName("design_model.proxy.jdk.Car").getMethod("run");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}

```