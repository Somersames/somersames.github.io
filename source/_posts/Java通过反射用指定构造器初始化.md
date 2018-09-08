---
title: Java通过反射用指定构造器初始化
date: 2018-03-12 14:02:50
tags: java高级特性
categories: Java
---
# Java通过反射用指定构造器初始化

首先， 一般来讲在Java中初始化一个类是通过new来操作的， 但是有一种情况却不适合这种new操作，那就是通过配置文件来进行实例化操作。

例如，在Spring中，需要加载配置文件中的类，这是比较常见的配置。  那么在Spring启动类中如何将这个类加载进容器中呢，显然进行new操作是不太现实的。 这时候就需要Java的反射操作了，Java的反射操作一般来讲有两种：分别是`Class.forName()`和`classLoader.loadCLass()` 最后都是通过`newInstance()` 来进行初始化，但是在这里却发现假设反射的类中含有带参数的构造器，那么此时这个newInstance()就会抛`NoSuchMethodException` ，这是因为newInstance()因为不加参数所以调用的是默认构造器，而反射类中已经包含了带参数的构造器，所以无不带参数构造器，遂抛出异常。

但是此时newInstance()是加不了参数的，所以若需要通过制定构造器来进行反射的话需要一个类叫Constructor，

新建一个实体类：

```
package reflec.cglib_test.ConstructTest;
public class Entity1 {
    static {
        System.out.println("static初始化");
    }

    public Entity1() {
    }

    int id;
    String name;
    public Entity1(int id, String name) {
        this.id = id;
        this.name = name;
    }
    public void say(){
        System.out.println(this.id);
        System.out.println(this.name);
    }
}

```

反射测试：

```
package reflec.cglib_test.ConstructTest;

import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;

public class Test1 {
    public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException, ClassNotFoundException {
//        Entity1 en2=Entity1.class.getConstructor(int.class,String.class).newInstance(1,"2");
//        Class.forName("reflec.cglib_test.ConstructTest.Entity1").newInstance();
//        Test1.class.getClassLoader().loadClass("reflec.cglib_test.ConstructTest.Entity1");
//        en2.say();
    }

    private static void do1(Object object){
        System.out.println(object.getClass().getName().toString());
    }
}

```

在上面的Test1中，下面两行在newInstance()中添加参数的话是会提示出错的。那么如何调用哪个带参数的构造器呢？ 这就是`Constructor` 类的功能了，他可以通过Class.getConstructor()来选择参数，在这里需要注意的是int及其他的java基本数据类型都是原生的类，非封装类。之后再newInstance()中输入参数既可以反射调用了。





