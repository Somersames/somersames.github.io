---
title: 对Java的List转Array的源码一点思考
date: 2018-04-07 19:42:48
tags: [java基础]
categories: Java
---
在Java里面，List转为Array是调用的Java的一个Arrays.copyOf()这个方法，查看了下源代码：
```java
public Object[] toArray() {
        return Arrays.copyOf(this.elementData, this.size);
    }

 public static <T> T[] copyOf(T[] original, int newLength) {
        return (T[]) copyOf(original, newLength, original.getClass());
    }

 public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
        @SuppressWarnings("unchecked")
        T[] copy = ((Object)newType == (Object)Object[].class)
            ? (T[]) new Object[newLength]
            : (T[]) Array.newInstance(newType.getComponentType(), newLength);
        System.arraycopy(original, 0, copy, 0,
                         Math.min(original.length, newLength));
        return copy;
    }


 public static Object newInstance(Class<?> componentType, int length)
        throws NegativeArraySizeException {
        return newArray(componentType, length);
    }
```

在这里需要注意的是在copyOf()方法中的那个三元表达式，也就是说在这里无论执行的true还是false，都会返回一个新的数组对象。而在这里有一行代码就是`((Object)newType == (Object)Object[]).class)，这一句看起来没什么，其实可以看参数`Class<? extends T[]> newType`和`Object`会发现这两个数组对象都被强转成了`Object`，而`==`比较符是不能比较不同类型的。例如：
```java
public void te(){
        System.out.println("ad" == 2);
        System.out.println((String.class == Object.class));
    }
```

上面的代码在代码的编译期就会被提示`==`不可以适用于上述的两种情况.如下：
![](不可转截图.png)

那么回到正题，这里其实仔细看就会发现无论是否相等，在这里三元表达式中，都会返回一个新的数组对象，那么在这里加入了三元表达式的想法是`newInstance()`是通过反射进行创建的，而`new Object[]`则是非反射创建的.
> Because reflection involves types that are dynamically resolved, certain Java virtual machine optimizations can not be performed. Consequently, reflective operations have slower performance than their non-reflective counterparts, and should be avoided in sections of code which are called frequently in performance-sensitive applications.


所以在这里加一个判断使代码尽量通过非反射的方式进行创建。最后调用`System.arraycopy()`,但是这里的都已经强转成了Object，应该不会有失败的情况