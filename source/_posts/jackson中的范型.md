---
title: jackson中的范型
date: 2021-04-16 23:27:27
tags: [Spring]
---
在 jackson 将字符串转为对象的时候，如果是不带有范型的数据类，那么在 strig 转 obj 的时候不会出现什么问题，如果你的对象带有范型的话，那么就需要注意了，稍不注意就会抛出如下异常
> java.util.LinkedHashMap cannot be cast to XX

出现该原因的原因在于 jackson 转换对象的时候，如果没有识别到原始类型，会默认将其转为 LinkedHashMap，后续一旦使用该类，就会抛出上述错误
demo如下：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/typeReference_in_jackson/Food.png)
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/typeReference_in_jackson/Apple.png)

如果你是使用如下方法进行 string 转 object，那么范型会被映射称为 `LinkedHashMap`
```java
public <T> T readValue(String content, Class<T> valueType)
```
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/typeReference_in_jackson/Food_strToObj.png)



# 一、jackson 是如何赋值的（默认构造器和多参数构造器）
## 无参构造器
对于含有无参数构造器的对象，那么处理流程很简单，就是直接通过 `_defaultCreator.call()` 来进行对象的创建，通过 `newInstance()` 来实例化，最后通过反射 invoke 方法进行赋值

## 有参数构造器

如果含有多个构造函数，那么就必须指明需要用哪一个构造函数进行初始化，否则代码会直接报错
> no Creators, like default constructor, exist

很明显，这个表示没有指定一个构造器作为 Jackson 实例化使用，因为对于多参数构造器，Jackson 不知道用赋值的顺序，所以需要人为进行指定
但是如果是两个 String 的入参，那么如何进行字段映射？，其实 jackson 还是不知道

这个时候就需要指定一个构造器了`@JsonCreator`，配合 `@JsonProperty` 指定字段。
因为对于多个参数的构造器，需要明确告诉 Jackson 如何对其进行赋值，否则会抛出以下错误
> Argument #0 of constructor [constructor for `xyz.somersames.model.Producer` (3 args)

```java
@JsonCreator
public Producer(@JsonProperty("name") String name, @JsonProperty("age") String age, @JsonProperty("friend") T friend) {
    this.name = name;
    this.age = age;
    this.friend = friend;
}
```

# 二、范型擦除
如果你在代码中这样写，在编译期间都是不通过的
```java
private void testType(){
    Food<Apple> food = new Food<Apple>();
    Food<LinkedHashMap> linkedHashMapFood = new Food<LinkedHashMap>();
    food = linkedHashMapFood;
}
```
那么为什么在运行期间可以将 LinkedHashMap 赋值到 food 上去呢，这就是范型擦除，在运行的时候 JVM 其实是不知道你的范型类型的
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/typeReference_in_jackson/Food_strToObj.png)

但是在 Jackson 是可以实现范型的转换，那么就要在运行期间获取到该类的原始类型，官方推荐如下方法：
```java
public <T> T readValue(String content, TypeReference<T> valueTypeRef)

//demo
objectMapper.readValue("XXX", new TypeReference<XXX>() {});

```
TypeReference 是一个抽象方法，如果你的 java 基础还可以的话，一眼就可以看到这就是一个匿名内部类，匿名内部类其实可以带有范型的原始信息的

## 匿名内部类
如果一个类中含有匿名内部类，那么在编译以后会形成两个 class 文件的，demo 如下：

![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/typeReference_in_jackson/TestA.png)

在编译以后会生成两个文件
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/typeReference_in_jackson/class_TestA.png)
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/typeReference_in_jackson/class_TestA1.png)

很明显，第二个 class 文件是带有原始的信息的，这说明匿名内部类是可以继承自原始类型的
> PS：这一点其实和 JDK 的动态代理类似，都是通过新生成一个 class 文件，但是作用却是不同的

## JackSon 获取原始类型

下面我以一个简单的例子来表示这两者的区别：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/typeReference_in_jackson/TestB.png)

打印结果如下：
![](https://szhtc-1252780558.cos.ap-shanghai.myqcloud.com/%E6%96%87%E7%AB%A0/typeReference_in_jackson/TestB_Print.png)

注意红框内部的区别，可以看到 ListWithInit 里面是一个匿名内部类，而上面所说了匿名内部类是会重新生成一个class文件的，从而导致 `ListWithInit` 是可以拿到范型信息的，
把 B 编译以后生成的 B$1.class 代码如下
```java
import java.util.ArrayList;

class B$1 extends ArrayList<String> {
    B$1(B var1) {
        this.this$0 = var1;
    }
}
```
可以看到编译后的 class 文件实际上已经把原始的范型信息包含了，而普通的 ArrayList 因为其父类还是 `AbstractList<E>`，所以在运行期间是拿不到原始的范型信息的

最后在 JDK 中通过 `getActualTypeArguments` 就可以拿到范型了

# 三、反射设置范型
```java
private void invokeTest() throws Exception {
    Class clazz = Class.forName("xyz.somersames.demo.Food");
    Food<Apple> food = new Food<Apple>();
    Method m1 = clazz.getDeclaredMethod("setT",Object.class);
    m1.invoke(food,new LinkedHashMap<String,String>());
    System.out.println(food.getT());
}
```

# 总结
其实说了那么多，Jackson 的转换大致流程就是先通过构造器来进行对象的实例化，最后通过反射进行字段的赋值操作