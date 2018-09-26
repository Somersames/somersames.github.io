---
title: 一次SpringBoot初始化引起的思考
date: 2018-09-25 23:03:38
tags: [SpringBoot]
categories: SpringBoot
---
在Spring中，经常会使用`@Resource`注解来自动装配一些Bean，但是在初始化的时候还是有一点小坑的，下面是一段代码，有三个类，分别是A，B，C。

类A：
```java
@Service
public class A {

    @Resource
    B b;


    public void someMethod(){
        System.out.println(b.getC() == null);
    }
}
```

类B：
```java
@Service
public class B {

    @Resource
    C c ;

    public C cParam =getC();

    public C getC(){
        return c.getAnewC();
    }

}
```

类C：
```java
@Service
public class C {
    private int i;

    private String name ;
    public C getAnewC(){
        C c =new C();
        c.setI(1);
        c.setName("a");
        return c;
    }

    public int getI() {
        return i;
    }

    public void setI(int i) {
        this.i = i;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}

```

此时这三个类有一个地方会抛出一个空指针异常，如果你可以一眼看出来的话，不妨继续走下去看看是否正确。
编写一个测试类，如下：
```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = DemoApplication.class)
public class SpringTest {


    @Autowired
    A a ;


    @Test
    public void hello(){
        a.someMethod();
    }
}

```

这样看下去，可能就看不出哪里有问题，但是运行之后会抛出一个`NullException`，错误日志如下:
```java
Caused by: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'b' defined in file [C:\Users\SZH\IdeaProject\firstcloud\target\classes\szh\demo\test\B.class]: Instantiation of bean failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [szh.demo.test.B]: Constructor threw exception; nested exception is java.lang.NullPointerException
--------------------------------------------------

Caused by: org.springframework.beans.BeanInstantiationException: Failed to instantiate [szh.demo.test.B]: Constructor threw exception; nested exception is java.lang.NullPointerException
------------------------------------------------------

Caused by: java.lang.NullPointerException
	at szh.demo.test.B.getC(B.java:21)
	at szh.demo.test.B.<init>(B.java:18)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
	at org.springframework.beans.BeanUtils.instantiateClass(BeanUtils.java:142)
	... 55 more
```
此时的异常栈如下，可以看到在类`B`里面，第18行，也就是`public C cParam =getC();`抛出了一空指针异常，这个异常的原因就是在方法`getC()`里面，c是一个null，那么在这里可能就会有一个疑问了，Spring不是会自动装配有`Resource`注解的吗？那么此时的`c`为什么没有被初始化。


## Spring的初始化
大家都知道，无论Spring无论怎样初始化，都是需要生成一个对象的，这个对象不管是通过`Class.loadClass`或者`Class.forName`等，那么也是无论逃不过Java的初始化，在这里为了验证`B`中的异常是在初始化B的时候产生的，此时修改B位如下：
```java
@Service
public class B {

    @Autowired
    C c ;
    {
        System.out.println("public C cParam =getC()上面一行");
    }
    public C cParam =getC();
    {
        System.out.println("public C cParam =getC()下面一行");
    }
    public C getC(){
        return c.getAnewC();
    }

}
```

在此调用测试类，你会发现`"public C cParam =getC()上面一行"`打印出来之后马上就会出错，这也印证了前面的猜想，这个异常是在初始化B的时候产生的。

### 猜想
那么在这里就基本上可以才出来Spring的一个加载流程了，首先Spring会扫描所有的带有注解得类，然后当初始化完毕之后放入Beanfactory，最后再进行一个变量的赋值，为了验证此猜想，修改`B`和`A`代码为如下：
类B
```java
@Service
public class B {

    @Autowired
    C c ;
    {
        System.out.println("public C cParam =getC()上面一行");
    }
    public C cParam =null;
    {
        System.out.println("public C cParam =getC()下面一行");
    }
    public C getC(){
        return c.getAnewC();
    }

}
```


类A：
```java
public class A {

    @Resource
    B b;

    public void someMethod(){
        System.out.println(b.getC() == null);
    }
}

```
此时解决。

## 总结
所以以后在初始化的时候，需要注意变量的初始化是否会涉及到一些对象