---
title: Java8中的流式简单总结
date: 2019-11-22 00:03:01
tags: Java
categories: Java
---
在大量使用Java8中的流式操作之后，觉得用起来还挺舒服的，所以正好趁这个机会总结下。

使用Java8的lambda表达式的时候，需要先把集合转为一种流，也就是调用 stream 方法，但是 stream 却是 Collection 类里面的一个方法，也就是只有 Collection 的子类才可以使用，所以 Map 集合是使用不了的，同理，对于数组，可以通过`Arrays.stream()` 方法来讲数组转为一个Stream，这样也可以使用Stream里面的方法了，

## 将集合转为流

下面介绍几个很常用的方法来介绍流式操作的便捷性。

```java
List<String> stringList = new ArrayList<>();
stringList.add("a1");
stringList.add("b1");
stringList.add("a2");
stringList.add("b2");
stringList.stream();
```

这样我们就可以得到了一个流式操作，接下来就可以使用Stream里面定义好的方法了



### filter

该方法用于过滤我们设置的一些判断条件，如下：

```java
private void streamFilter(List<String> list){
        List<String> result = list.stream().filter(item ->{
            if("a".equals(item)){
                return true;
            }
            return false;
        }).collect(Collectors.toList());
    }
```

这个只是一个普通的String数组遍历，可以看到如果我们通过Java8之前的代码写的话，会首先 new一个List，然后通过add方法来进行插入，这样虽然不会出现什么问题，但是会显得不是那么整洁，但是用流式操作的话，感觉会方便不少。

如果我们有多个条件要进行过滤的话，filter也是支持链式调用的。

```java
private void streamFilter(List<String> list){
        List<String> result = list.stream().filter(item ->{
            if("a1".equals(item)){
                return true;
            }
            return false;
        })
        .filter(item -> (item.startsWith("a")))
        .collect(Collectors.toList());
        result.stream().forEach(item -> System.out.println(item));
    }
```

### map

map常用于一些遍历条件，例如取出某些JavaBean的属性并作为集合。

```java
public class Person {
    private String name;
    private Integer age;
    private String sex;
}
```

将一个Person集合的所有姓名取出：

```java
//取出集合内所有的姓名
List<String> nameList = list.stream().map(Person::getName).collect(Collectors.toList());

//去重取出集合内所有的姓名
List<String> nameDistinctList = list.stream().map(Person::getName).distinct().collect(Collectors.toList());

//找出所有成年的人
List<String> ageList = list.stream().filter(
                item -> (Objects.nonNull(item.getAge()) && item.getAge() > 18)
        ).map(Person::getName).distinct().collect(Collectors.toList());

//将所有已经成年的人按照年龄进行分组
Map<Integer, List<Person>> ageSameList = list.stream().filter(
                item -> (Objects.nonNull(item.getAge()) && item.getAge() > 18)
        ).collect(Collectors.groupingBy(Person::getAge));
```

### findFirst

在我的使用过程中，感觉这个方法配合枚举类相当的好用：

```java
public enum ProvinceEnum {
    BEIJING("北京市","110000"),
    TIANJIN("天津市","120000"),
    HEBEI("河北省","130000"),
    SHANXI("山西省","140000"),
    ;
    private String cn;
    private String code;

    ProvinceEnum(String cn, String code) {
        this.cn = cn;
        this.code = code;
    }

    public String getCn() {
        return cn;
    }

    public String getCode() {
        return code;
    }

    public static String getCnByCode(String code){
        return Arrays.stream(ProvinceEnum.values()).filter(
                item -> (StringUtils.isNotEmpty(item) && item.getCn().equals(code))
        ).map(ProvinceEnum::getCn).findFirst().orElse(null);
    }
}
```
 由于其他的方法大致与这上面几个方法相似，所以就不再写demo了