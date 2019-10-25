---
layout: post
title:  "Stream API——Lambda表达式的核心包"
categories: Java
description: Stream API——Lambda表达式的核心包。
keywords: Lambda, lambda, Stream, Stream API, Java8, JAVA8
---

# Lambda Of StreamAPI

## 一、Lambda表达式

### 1.1 What ###
**Lambda表达式**只是一个匿名函数。什么是匿名函数？
```js
//JS匿名函数示例
(function(x, y){
    alert(x + y);  
})(2, 3);
```
作为常量值，没有名称的函数，通常作为其他函数的参数。

典型的lambda表达式语法如下：
```java
(x, y) -> x + y //该函数有两个参数x、y,并且返回他们的和。
```
lambda表达式最重要的特性是它们是在上下文中执行的。所以以上两个参数x、y可以匹配整数，也可以匹配字符串，根据上下文的不同自动匹配。

### 1.2 基本语法 ###

```java
// 1、
(parameters) -> expression
// 2、
(parameters) -> { statements; }
// 3、
() -> expression
```

eg:
```java
(int a, int b) ->    a * b
(a, b)          ->   a - b
() -> 99
(String a) -> System.out.println(a)
a -> 2 * a
c -> { //some complex statements }
```

编写lambda表达式的**规则**：
1. 可以包含零个，一个或多个参数；
2. 可以显式声明参数的类型，也可以从上下文中推断出参数的类型；
3. 多个参数括在强制括号中，并以逗号分隔，空括号用于表示一组空参数；
4. 当存在单个参数时，如果推断其类型，则不必使用括号；
5. lambda表达式的主体可以包含一个或多个语句；
6. 如果lambda表达式的主体具有单个语句，则大括号不是必需的，并且匿名函数的返回类型与正文表达式的返回类型相同。当正文中有多个语句时，必须用大括号括起来。

### 1.3 函数式接口(Functional Interface) ###

首先，我们需要了解一下**单个抽象方法接口Single Abstract Method Interfaces(SAM Interfaces)**，即只有一种抽象方法的接口。JAVA中已经有很多这样的SAM接口，例如Runnable。Java8通过新注解(**@FunctionalInterface**)标记这些接口来强制执行**单一职责**规则。
```java
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
```

我们知道lambda表达式是匿名函数，大多数是作为**参数**传递给其他函数的。在JAVA中，函数的**参数**总有一个类型，因此，基本上每个lambda表达式必须转换为某种类型才能成为其他函数的**参数**。给个例子：
```java
new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("YazidChen");
    }
}).start();

//lambda表达式形式
new Thread(() -> System.out.println("YazidChen")).start();
```
上面的例子中，lambda表达式被转化为Runnable类型。

### 1.4 相关例子 ###

1、迭代List：
```java
List<String> pointList = new ArrayList();
pointList.add("1");
pointList.add("2");

pointList.forEach(p -> {
            System.out.println(p);
            //Do more work
            });
//or
pointList.forEach(System.out::println);
```

2、创建一个新的runnable并将其传递给线程
```java
new Thread(() -> System.out.println("YazidChen Runnable")).start();
```

## 二、Streams ##

尝试将集合流化，java.util.Stream可以执行一个或多个操作的流。Stream的特性有：
- 不是数据结构
- 专为lambda设计
- 不支持索引访问
- 可以轻松输出为数组或列表
- 支持延迟访问
- 可并行

eg:
```java
List<Integer> numbers = Arrays.asList(1, 2, 3);
numbers.stream().mapToInt(n -> n * n).sum();
```
Stream处理过程分为三个部分，创建、转换、聚合：
![](https://yazid-public.oss-cn-shenzhen.aliyuncs.com/blog/images/20181022114451.png?x-oss-process=style/Watermark)

### 2.1 构建流 ###

```java
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.stream.IntStream;
import java.util.stream.Stream;

/**
 * @author YazidChen
 * @date 2018/10/04 0004 17:10
 */
public class StreamDemo {

    private void streamOf() {
        Stream<Integer> streamA = Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9);
        //equals
        Stream<Integer> streamB = Stream.of(new Integer[]{1, 2, 3, 4, 5, 6, 7, 8, 9});

        streamA.forEach(System.out::println);
        //equals
        streamA.forEach(p -> System.out.println(p));
    }

    private void listStream() {
        List<Integer> list = new ArrayList<>();
        for (int i = 1; i < 10; i++) {
            list.add(i);
        }

        Stream<Integer> stream = list.stream();
        stream.forEach(System.out::println);
        //equals
        list.forEach(System.out::println);
    }

    private void strStream(){
        IntStream streamA = "12345_abcdefg".chars();
        streamA.forEach(System.out::println);

        Stream<String> streamB = Stream.of("A,B,C,D".split(","));
        streamB.forEach(System.out::println);
    }

    private void streamGenerate() {
        Stream<Date> stream = Stream.generate(Date::new).limit(10);
        stream.forEach(System.out::println);
    }

    private void streamIterate() {
        Stream.iterate(1, i -> i + 1).limit(10).forEach(System.out::println);
    }

    private void streamConcat() {
        Stream<Integer> stream1 = Stream.of(1, 2, 3);
        Stream<Integer> stream2 = Stream.of(4, 5);
        Stream.concat(stream1, stream2).forEach(System.out::println);
    }

    private void streamBuilder() {
        Stream<Object> stringStream = Stream.builder()
                .add("1")
                .add("2")
                .build();
    }
}
```

### 2.2 将流转换为集合 ###

```java
    private void streamToList() {
        List<Integer> list = new ArrayList<>();
        for (int i = 1; i < 10; i++) {
            list.add(i);
        }
        Stream<Integer> stream = list.stream();
        List<Integer> evenNumbersList = stream.filter(i -> i % 2 == 0).collect(Collectors.toList());
        System.out.print(evenNumbersList);
    }

    private void streamToArray() {
        List<Integer> list = new ArrayList<>();
        for (int i = 1; i < 10; i++) {
            list.add(i);
        }
        Stream<Integer> stream = list.stream();
        Integer[] evenNumbersArr = stream.filter(i -> i % 2 == 0).toArray(Integer[]::new);
        System.out.print(evenNumbersArr);
    }
```

### 2.3 核心函数 ###

```java
    private void streamCore() {
        List<String> memberNames = new ArrayList<>();
        memberNames.add("Amitabh");
        memberNames.add("Shekhar");
        memberNames.add("Aman");
        memberNames.add("Rahul");
        memberNames.add("Shahrukh");
        memberNames.add("Salman");
        memberNames.add("Yana");
        memberNames.add("Lokesh");

        /*
          Stream中间操作
         */
        //filter()，过滤
        System.out.println("--filter:");
        memberNames.stream().filter(s -> s.startsWith("A"))
                .forEach(System.out::println);

        //map()，转换
        System.out.println("--map:");
        memberNames.stream().map(String::toUpperCase)
                .forEach(System.out::println);

        //flatMap，多层级扁平化转换
        System.out.println("--flatMap:");
        Stream<List<Integer>> intStream = Stream.of(Arrays.asList(1, 2, 3), Arrays.asList(4, 5), Arrays.asList(6, 7, 8));
        intStream.flatMap(Collection::stream).forEach(System.out::println);

        //sorted(),排序
        System.out.println("--sorted:");
        memberNames.stream().sorted()
                .forEach(System.out::println);

        //peek(),生成一个包含原Stream所有元素的新Stream，同时提供一个消费函数，与原Stream并行执行
        System.out.println("--peek:");
        long totalMatched = memberNames.stream()
                .peek(System.out::println)
                .count();
        System.out.println(totalMatched);

        /*
          Stream终端操作
         */
        //forEach()，遍历
        System.out.println("--forEach:");
        memberNames.forEach(System.out::println);

        //collect()，收集
        System.out.println("--collect:");
        List<String> memNamesInUppercase = memberNames.stream().sorted()
                .map(String::toUpperCase)
                .collect(Collectors.toList());
        System.out.print(memNamesInUppercase);

        //match()，匹配
        System.out.println("--match:");
        boolean matchedResult = memberNames.stream()
                .anyMatch(s -> s.startsWith("A"));
        System.out.println(matchedResult);

        matchedResult = memberNames.stream()
                .allMatch(s -> s.startsWith("A"));
        System.out.println(matchedResult);

        matchedResult = memberNames.stream()
                .noneMatch(s -> s.startsWith("A"));
        System.out.println(matchedResult);

        //count()，总数
        System.out.println("--count:");
        long count = memberNames.stream()
                .filter(s -> s.startsWith("A"))
                .count();
        System.out.println(count);

        //reduce()，指定计算模型生成值
        System.out.println("--reduce:");
        Optional<String> reduced = memberNames.stream()
                .reduce((s1, s2) -> s1 + "#" + s2);
        reduced.ifPresent(System.out::println);

        /*
          条件操作
         */
        //anyMatch()，任意匹配
        System.out.println("--anyMatch:");
        boolean matched = memberNames.stream()
                .anyMatch(s -> s.startsWith("A"));
        System.out.println(matched);

        //findFirst()，首条
        System.out.println("--findFirst:");
        String firstMatchedName = memberNames.stream()
                .filter(s -> s.startsWith("L"))
                .findFirst().get();
        System.out.println(firstMatchedName);

        //distinct()，去重
        System.out.println("--distinct:");
        Collection<String> list = Arrays.asList("A", "B", "C", "D", "A", "B", "C");
        List<String> distinctElements = list.stream().distinct().collect(Collectors.toList());
        System.out.println(distinctElements);

        //skip()，从第N位开始取数
        System.out.println("--skip:");
        IntStream.range(1, 100).skip(10).limit(10).forEach(System.out::println);

        //max()，最大值
        System.out.println("--max:");
        Integer maxNumber = Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9)
                .max(Comparator.comparing(Integer::valueOf))
                .get();
        System.out.println("maxNumber = " + maxNumber);
        String maxChar = Stream.of("H", "T", "D", "I", "J")
                .max(Comparator.comparing(String::valueOf))
                .get();
        System.out.println("maxChar = " + maxChar);

        //min()，最小值
        System.out.println("--min:");
        Integer minNumber = Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9)
                .min(Comparator.comparing(Integer::valueOf))
                .get();
        System.out.println("minNumber = " + minNumber);
        String minChar = Stream.of("H", "T", "D", "I", "J")
                .min(Comparator.comparing(String::valueOf))
                .get();
        System.out.println("minChar = " + minChar);

        List<Car> cars = new ArrayList<>();
        cars.add(new Car("Jeep", "Wrangler", 2011));
        cars.add(new Car("Dodge", "Avenger", 2010));
        cars.add(new Car("Buick", "Cascada", 2016));
        cars.add(new Car("BMW", "BMW1", 2011));
        cars.add(new Car("TSL", "TSL1", 2010));
        cars.add(new Car("BYD", "BYDTang", 2016));

        //groupingBy()，分组
        System.out.println("--groupingBy:");
        Map<Integer, List<Car>> groupCar = cars.stream().collect(Collectors.groupingBy(Car::getYear));
        System.out.println(groupCar);

        //partitioningBy()，分片
        System.out.println("--partitioningBy:");
        Map<Boolean, List<Car>> partitionCar = cars.stream().collect(Collectors.partitioningBy(c -> c.getYear() == 2016));
        System.out.println(partitionCar);

        //comparing(),比较
        System.out.println("--comparing:");
        Comparator<Car> comparator = Comparator.comparing(Car::getYear);

        Car minObject = cars.stream().min(comparator).get();
        Car maxObject = cars.stream().max(comparator).get();

        System.out.println("minObject = " + minObject);
        System.out.println("maxObject = " + maxObject);
    }
```

基本元素流盒装：
```java
    private void streamBoxed() {
        List<Integer> ints = IntStream.of(1, 2, 3, 4, 5)
                .boxed()
                .collect(Collectors.toList());
        System.out.println(ints);

        List<Long> longs = LongStream.of(1L, 2L, 3L, 4L, 5L)
                .boxed()
                .collect(Collectors.toList());
        System.out.println(longs);

        List<Double> doubles = DoubleStream.of(1d, 2d, 3d, 4d, 5d)
                .boxed()
                .collect(Collectors.toList());
        System.out.println(doubles);
    }
```

随机数：
```java
    private void randomStream() {
        Random random = new Random();

        //5个随机整数
        random.ints(5).sorted().forEach(System.out::println);

        //5个随机浮点数，取值范围0~0.5
        random.doubles(5, 0, 0.5).sorted().forEach(System.out::println);

        //5个长整型数
        random.longs(5).boxed().collect(Collectors.toList()).forEach(System.out::println);
    }
```

### 2.4 并行流parallelStream ###

如下示例可能得到任意的顺序：
```java
    private void parallelStream() {
        List<Integer> list = new ArrayList<>();
        for (int i = 1; i < 10; i++) {
            list.add(i);
        }
        //创建并行流
        Stream<Integer> parallelStream = list.parallelStream();
        Integer[] evenNumbersArr = parallelStream.toArray(Integer[]::new);
        System.out.print(Arrays.toString(evenNumbersArr));
        System.out.println();
        System.out.println(parallelStream.isParallel());
        //sequential()，并行转换为顺序
        Stream<Integer> stream = parallelStream.sequential();
        System.out.println(stream.isParallel());
    }
```


## 参考 ##

本文参考以下文章，在此对原作者表示感谢！

[java-8-tutorial](https://howtodoinjava.com/java-8-tutorial/)

[Java8 Stream API使用](https://my.oschina.net/u/2377110/blog/1573456)