---
layout: post
title:  "Java lambda 表达式和Stream"
date:   2017-08-29 11:51:05
categories: Java8
excerpt: Java lambda表达和Stream用法介绍。
---

* content
{:toc}

# Stream

* 终端操作：reduce,collect, forEach,count等
* 中间操作：map、limit、skip等

终端操作的返回值不是流，中间操作的返回值仍然是流

## 获取流对象

```java
// 将集合对象直接转换为stream
List<String> list = new ArrayList<>();
Stream<String> stream1 =  list.stream();
// 将数组转换为stream
String[] Strings = new String[] {"chen", "lin"};
Stream<String> stream2 = Arrays.stream(Strings);
// 直接生产stream
Stream<String> stream3 = Stream.of("a","b","c");
```



## 转化为List

```java
list.stream().collect(Collectors.toList());
```



## 转化为Map

要保证key是唯一的，否则会报错。

```java
System.out.println(list.stream().collect(Collectors.toMap(a -> a + " key", a -> a + " value")));
```



##转化为Set

```java
System.out.println(list.stream().collect(Collectors.toSet()));
```



## 计数

###count()

```
list.stream().count();
```



## 筛选

### 初始化一个Person对象

```java
public class Person
{
    private String name;
    
    private int age;
    
    public Person(String name, int age)
    {
        this.name = name;
        this.age = age;
    }
    
    public String getName()
    {
        return name;
    }
    
    public void setName(String name)
    {
        this.name = name;
    }
    
    public int getAge()
    {
        return age;
    }
    
    public void setAge(int age)
    {
        this.age = age;
    }
    
    @Override
    public String toString()
    {
        return name + ":" + age;
    }
    
    @Override
    public boolean equals(Object o)
    {
        if (this == o)
            return true;
        if (!(o instanceof Person))
            return false;
        
        Person person = (Person)o;
        
        if (getAge() != person.getAge())
            return false;
        return getName() != null ? getName().equals(person.getName()) : person.getName() == null;
    }
    
    @Override
    public int hashCode()
    {
        int result = getName() != null ? getName().hashCode() : 0;
        result = 31 * result + getAge();
        return result;
    }
}
```

### 初始化数据

```java
List<Person> personList = new ArrayList<>();
personList.add(new Person("zhangsan", 20));
personList.add(new Person("lisi", 21));
personList.add(new Person("wangwu", 22));
personList.add(new Person("wangwu", 22));
```

###filter()

```java
System.out.println(personList.stream().filter(p -> p.getName().equals("zhangsan")).collect(Collectors.toList()));
```

## 去重

###distinct()

```java
System.out.println(personList.stream().distinct().collect(Collectors.toList()));
```

## 截取

###limit()

```java
// 截取前几个
System.out.println(personList.stream().limit(2).collect(Collectors.toList()));
```

## 跳过

###skip()

```java
// 跳过几个
System.out.println(personList.stream().skip(2).collect(Collectors.toList()));
```



## 映射

###map()

```java
// 映射
// 从person转换为personName
System.out.println(personList.stream().map(person -> person.getName()).collect(Collectors.toList()));
```

### Collecting.mapping()

```java
System.out.println(personList.stream().collect(Collectors.mapping(p ->p, Collectors.toList())));
```



## 平面映射（汇集）

###flatMap()

```java
// 汇流成河。 stream里的对象（personList, personList1）本身就是可以提取stream的，提取出来汇集成更大的stream
System.out.println(Stream.of(personList, personList1).flatMap(a -> a.stream()).collect(Collectors.toList()));
```

##最大最小

### max()和min()

```java
// 最大最小
System.out.println(personList.stream().max((p, p1) -> p.getAge() - p1.getAge()).get());
```



## 排序

###sort()

```java
// 排序
System.out.println(personList.stream().sorted((a, b) -> b.getAge() - a.getAge()).collect(Collectors.toList()));
```



## 分组

### groupingBy()

```java
// 分组
System.out.println(personList.stream().collect(Collectors.groupingBy(p->p.getAge(), Collectors.mapping(p ->p, Collectors.toList()))));
```



## 遍历

### forEach()



```java
// 遍历
personList.stream().forEach(person -> {
    System.out.println(person);
});
```

##规约

### reduce()

reduce的第一个参数表示初始值

reduce的第二个参数为需要进行的归约操作，它接收一个拥有两个参数的Lambda表达式。第一个为总和，会传给下一个acc，第二个参数为当前元素

```java
System.out.println(personList.stream().reduce(new Person("",1),(acc,p)->new Person("",acc.getAge()+p.getAge())));
```

# Lambda

可以把Lambda表达式理解为简洁地表示可传递的匿名函数的一种方式：它没有名称，但它有参数列表、函数主体、返回类型

```java
//普通
(Apple a1, Apple a2) -> a1.getWeight().compareTo(a2.getWeight())
//参数隐藏
(a1, a2) -> a1.getWeight().compareTo(a2.getWeight())
//适用于多行
(Apple a1, Apple a2) -> {a1.getWeight().compareTo(a2.getWeight());}

//等价于
new MyInterface(){
  @Override
  public void method(Apple a1,Apple a2){
    return a1.getWeight().compareTo(a2.getWeight());
  }
}
```



###方法引用

方法引用让你可以重复使用现有的方法定义，并像Lambda一样传递它们。在一些情况下，比起使用Lambda表达式，它们更易读，感觉也更自然

| Lambda                                   | 方法引用                              |
| ---------------------------------------- | --------------------------------- |
| () -> Thread.currentThread().dumpStack() | Thread.currentThread()::dumpStack |
| (str,i) -> str.substring(i)              | String::substring                 |
| (String s) -> System.out.println(s)      | System.out::println               |
| (Apple a)->a.getWeight()                 | Apple::getWeight                  |




