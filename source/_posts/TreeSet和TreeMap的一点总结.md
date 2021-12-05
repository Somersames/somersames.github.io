---
title: TreeSet和TreeMap的一点总结
date: 2018-05-24 15:50:41
tags: [Java,数据结构]
categories: [Java,JDK]
---
首先简单介绍下TreeSet和TreeMap的两种排序：
* 自然排序
* 通过comparator排序
```java
private static void compareWithCpmparator(){
        TreeSet<String> treeSet =new TreeSet<>();
        List<String> list =new ArrayList<>();
        list.add("a");
        list.add("d");
        list.add("b");
        treeSet.addAll(list);
        Iterator<String> iterator =treeSet.iterator();
        while (iterator.hasNext()){
            System.out.println(iterator.next());
        }
        Comparator<String>  comparator1 = (Comparator<String>) treeSet.comparator();
        if (comparator1 == null){
            System.out.println("comparator1是空");
        }else {
            System.out.println("comparator1不是空");
        }
    }

    public static void main(String[] args) {
        compareWithCpmparator();
    }
```
运行之后的结果如下:
```java
a
b
d
comparator1是空
```
这段代码里面获取的`comparator`是空的，Debug一遍，发现这个方法其实调用的是`NavigableMap`里面的`comparator`
```java
    public Comparator<? super E> comparator() {
        return m.comparator();
    }
```
查看官网上对其的介绍：
```
Comparator<? super K> comparator()
Returns the comparator used to order the keys in this map, or null if this map uses the natural ordering of its keys.
Returns:
the comparator used to order the keys in this map, or null if this map uses the natural ordering of its keys
```
在调用这个方法的时候若是自然排序，那么会返回一个null。若是通过comparator进行排序的话当前集合采用的`comparator`。
查看官网对reeSet的无参构造器的解释：
>   /**
     * Constructs a new, empty tree set, sorted according to the
     * natural ordering of its elements.  All elements inserted into
     * the set must implement the {@link Comparable} interface.
     * Furthermore, all such elements must be <i>mutually
     * comparable</i>: {@code e1.compareTo(e2)} must not throw a
     * {@code ClassCastException} for any elements {@code e1} and
     * {@code e2} in the set.  If the user attempts to add an element
     * to the set that violates this constraint (for example, the user
     * attempts to add a string element to a set whose elements are
     * integers), the {@code add} call will throw a
     * {@code ClassCastException}.

在使用TreeSet的时候，插入的元素需要实现Comparable这个接口，而刚刚的元素是String，查看String的代码发现:

```java
public final class String implements java.io.Serializable, Comparable<String>, CharSequence {
```
确实实现了，再测试一个没有实现的元素：
```java
public class PojoTest {
    private int id;
    private String name;

    public PojoTest() {
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public PojoTest(int id, String name) {
        this.id = id;
        this.name = name;
    }
}

 private static void  com(){
        TreeSet<PojoTest> treeSet =new TreeSet<>();
        treeSet.add(new PojoTest(1,"a"));
        treeSet.add(new PojoTest(2,"b"));
        treeSet.add(new PojoTest(3,"c"));
        Iterator<PojoTest> iterator =treeSet.iterator();
        while (iterator.hasNext()){
            System.out.println(iterator.next().getName());
        }
    }
```

运行结果如下:
```java
Exception in thread "main" java.lang.ClassCastException: SetAndMap.TreeSetAndTreeMap.PojoTest cannot be cast to java.lang.Comparable
	at java.util.TreeMap.compare(TreeMap.java:1294)
	at java.util.TreeMap.put(TreeMap.java:538)
	at java.util.TreeSet.add(TreeSet.java:255)
	at SetAndMap.TreeSetAndTreeMap.TestTreeSet.com(TestTreeSet.java:77)
	at SetAndMap.TreeSetAndTreeMap.TestTreeSet.main(TestTreeSet.java:88)
```
很明显，所以放在TreeSet里面的元素要么是实现Comparable了的自然排序，要么是通过comparator来进行排序的。

**最后附上一个标准的使用Comparator的方法**：
```java
private static void  construct(){
        Comparator<String>  comparator =new Comparator<String>() {
            @Override
            public int compare(String o1, String o2) {
                if(o1.toCharArray()[0] >o2.toCharArray()[0]){
                    return -1;
                }else if(o1.toCharArray()[0] == o2.toCharArray()[0]){
                    return 0;
                }else{
                    return 1;
               }
            }
        };
        TreeSet<String> treeSet =new TreeSet<>(comparator);
        List<String> list =new ArrayList<>();
        list.add("a");
        list.add("d");
        list.add("b");
        treeSet.addAll(list);
        Iterator<String> iterator =treeSet.iterator();
        while (iterator.hasNext()){
            System.out.println(iterator.next());
        }
        Comparator<String>  comparator1 = (Comparator<String>) treeSet.comparator();
        TreeSet<String> treeSet1 =new TreeSet<>(comparator1);
        treeSet1.add("c");
        treeSet1.add("g");
        treeSet1.add("a");
        Iterator<String> iterator1 =treeSet1.iterator();
        while (iterator1.hasNext()){
            System.out.println(iterator1.next());
        }
```

## 自然排序：
是实现Comparable接口并且重写了compareTo方法的
## 另一个comparator
则是通过comparator并且重写compare方法