# lambda表达式

Lambda表达式是JDK8中出现的新特性，其是函数接口的一种实现方式，用于代替匿名内部类

## 概念

函数式接口，Functional Interface，也称为功能性接口。简单来说，接口中可以包含多个方法，但是仅能有一个自己的抽象方法

建议加上@FunctionalInterface，相当于加一个约束，团队其他若想加抽象方法则会报错

## 内置函数式接口

| 内置接口          | 参数 | 返回值  | 描述                         |
| ----------------- | ---- | ------- | ---------------------------- |
| Predicate<T>      | T    | boolean | 断言                         |
| Consumer<T>       | T    | -       | 仅消费一个数据，没有返回值   |
| Supplier<T>       | -    | T       | 仅提供一个数据，无需要输入   |
| Function<T,R>     | T    | R       | 仅输入一个数据，返回一个数据 |
| UnaryOperator<T>  | T    | T       | 输入输出相同的Function       |
| BiFunction<T,U,R> | T,U  | R       | 输入两数据，返回一个数据     |
| BinaryOperator<T> | T,T  | T       | 输出输出都相同的             |

### Predicate<T>

```java
Predicate<Integer> pre = i -> i>7;
System.out.println(pre.test(8));//true
System.out.println(pre.test(6));//false
IntPredicate intPre = i -> i < 3;
//还可以是其他类型
//测试默认方法
Predicate<Integer> gt8 = i -> i>8;
Predicate<Integer> lt8 = i -> i<3;
System.out.println(gt8.and(lt3).test(9))//false
System.out.println(gt8.or(lt3).test(9))//true
System.out.println(gt8.negate().test(9))//false
//equals比较
System.out.println(Predicate.isEquals("Hello").test("hello"))
```

### Consumer<T>

消费，为什么叫消费者，因为它只有输入没有输出，所以称之为消费者

```java
Consumer<String> con = str -> System.out.println("hello, " + str);
con.accept("tome");
Consumer<Integer> con1 = n -> System.out.println(n * 2);
Consumer<Integer> con2 = n -> System.out.println(n * n);
con1.andThen(con2).accept(5);
```

### Supplier<T>

提供者，没有输入，有返回值

```java
Supplier<String> supp = () -> "Lambda";
System.out.println(supp.get());
```

### Function<T,R>

一个参数一个返回值

有三个默认方法

```java
default <V> Function<V, R> compose(Function<? super V, ? extends T> before) {
    Objects.requireNonNull(before);
    return (V v) -> apply(before.apply(v));
}
default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
    Objects.requireNonNull(after);
    return (T t) -> after.apply(apply(t));
}
static <T> Function<T, T> identity() {
    return t -> t;
}
Function<Integer,String> fun = n -> "i love you：" + n;
System.out.println(fun.apply(2019));
Function<Integer,Integer> fun1 = n -> n*2;
Function<Integer,Integer> fun2 = n -> n*n;
//先将5作为fun1的参数，计算结果为10
//再将fun1的计算结果10作为fun2的参数再计算
System.out.println(fun1.andThen(fun2).apply(5));//100
//将5作为fun2的参数，计算结果25
//再将fun2的结果25作为fun1的参数，计算结果为50
System.out.println(fun1.compose(fun2).apply(5));//50
Function.identity.apply(5);//5,参数放什么结果就是什么
```

### UnaryOperator<T>

输入输出类型相同的Function，Function能用的，这里都能用 UnaryOperator<T> extends Function<T,T>

### BiFunction<T,U,R>

三个泛型，前两是输入参数类型，第三个泛型是返回值类型

## lambda方法引用

可以借助内置函数式接口对一个类中的静态方法，实例方法，构造方法进行使用的方式

```java
public class Person{
				
    private String name;
    public Person(){}

    public Person(String name){
        this.name = name;
    }
    //静态方法
    public static void sleeping(int hours){
        System.out.println();
    }
    //实例方法
    //在实例方法的第一个参数位置存在一个隐藏参数this
    //这就是为什么可以在方法内使用this，是因为隐藏了一个this参数
    public String play(int minutes){
        return name + "已经玩儿了" + minutes + "分钟了";
    }

    //实例方法
    public void study(String course){
        System.out.println()
    }
}
//测试
//传统方式
Person person = new Person("张三");
person.play(5);
person.study("ss");
//如果在实例方法的参数里显示的添加参数
public String play(Person this,int minutes){
    return name + "已经玩儿了" + minutes + "分钟了";
}
//实例方法
public void study(Person this,String course){
    System.out.println()
}
//以上的测试方法也可以运行，不会报错
person.play(5);
person.study("ss");
//注：只对实例方法才有效
//Lambda静态方法引用，引用方式：类名 :: 方法名
//sleeping(int hours)方法一个输入没有输出，符合函数式接口Consumer的定义
Consumer<Integer> consumer = Person :: sleeping;
consumer.accept(8);//相当于，Person.sleeping(8);
//Lambda实例方法引用   实例名 :: 方法名
Person person = new Person("李四");
//play方法，有一个输入一个输出，符合Function的定义
Function<Integer,String> function = person :: play;
function.aplly(5);//相当于，person.play(5);
//加上this的参数，两输入一个输出，符合BiFunction的定义
BiFunction<Person, Integer, String> bf = Pserson :: play;
bf.apply(person,5);
//以上说明了为什么实例方法的引用也可以用类名加方法名，实际的意义是调用的，方法有一个隐藏的this参数
//study方法只有一个输入没有输出，符合consumer的定义
Consumer<String> con = person :: study;
con.accept("webflux");
//Lambda无参构造器引用   类名 :: new
//无参构造器，没有输入一个输出，符合函数式接口Supply的定义
Supplier<Person> supp = Person :: new;
Person person = supp.get();//相当于new person
//代参构造器
Function<String,Person> fun = Pserson :: new;
Person person = fun.apply("王五") // 相当于 new Person("王五");
```



# Stream流编程

Stream流编程是一种函数式编程

## Stream概述

### 函数式编程

函数式编程是一种编程的范式，是如何编写程序的一种方法论。它属于结构化编程的一种，主要思想是把对数据的一连操作通过链函数调用的方式来实现。其属于Fluent风格编程的一种，具有异曲同工的效果。

函数式编程属于Fluent风格的一种情况，总结来说，Fluent风格范围大于函数式编程的范围

### Stream简介

JDK8中，Stream是一个接口，其中包含了很多的方法，这些方法可以对流中的数据进行操作，这些方法分为三类

- 创建Stream接口对象的方法，这些方法一般都需要通过输入原始操作数据，然后创建数据池。
- 返回Stream类型的中间操作方法
- 返回其他类型的方法。这些方法即终止操作方法

### Stream与函数式编程

Stream流编程是一种函数式编程，或者说，函数式编程是Stream流编程的实现方式。

Stream流编程的具体实现步骤如下：

1. 输入要操作的数据，将数据放入到数据池，而Stream就是这个数据池
2. 调用对象接连调用中间操作函数，这些函数均返回调用者对象类型。
3. 调用终止操作函数，是链调用结束，让其返回最终的操作结果，即数据池中最终数据

```java
int[] nums = {1,2,3};
int sum = IntStream.of(nums)
    .map(p -> p*2 )//变成了{2,4,6}，中间操作，返回stream
    .map(p -> p*p)//变成了{4,16,36}，中间操作，返回stream
    .sum();//返回值是整形，所以这个就是终止操作
System.out.println(sum);
int sum = IntStream.of(nums)
    .map(p -> p*2 )//变成了{2,4,6}，中间操作，返回stream
    .map(p -> {
        System.out.println(p + "进行乘方");
        return p*p;
    })//变成了{4,16,36}，中间操作，返回stream
    .sum();//返回值是整形，所以这个就是终止操作
//静态方法
public static int compute(int n){
    return n*n;
}
int sum = IntStream.of(nums)
    .map(p -> p*2 )//变成了{2,4,6}，中间操作，返回stream
    .map(Test :: compute)//变成了{4,16,36}，中间操作，返回stream
    .sum();//返回值是整形，所以这个就是终止操作
```

### 惰性求值

Stream流编程中对于中间方法的执行，存在一个惰性求值的问题，多个中间操作可以连起来形成一个链调用，除非链调用的最后执行了终止方法，否则中间操作是不会执行任何处理的。即只有当终止操作执行时才会触发链调用

```java
IntStream intStream = IntStream.of(nums)
    .map(p -> p*2 )//变成了{2,4,6}，中间操作，返回stream
    .map(Test :: compute);//变成了{4,16,36}，中间操作，返回stream
//操作不会执行，因为没有执行终止方法
```

Stream流编程类似于生产流作业，一条流水线只能向前，不会回退，最终得到产品

惰性求值的意思是，如果最后一个包装的工人没有上岗，前面的员工也不能工作，因为不能形成最终的产品

Stream流编程需要注意的问题：

- 中间操作返回的都是Stream，可以链调用
- 这些函数都有一个共同的任务：对stream流中的每一个数据都进行操作
- 一个中间函数对于数据池的操作结果将作为下一个中间函数的数据输入
- 所有操作都是不可逆转的，一旦操作就会影响数据池中的数据

### Stream流的创建

```java
//使用数组创建流
int[] nums = {1,2,3};
IntStream stream = IntStream.of(nums);
IntStream stream = Arrays.stream(nums);
//使用集合创建流
List<String> names = new ArrayList<>();
Stream<String> stream = names.stream();
//数字流
IntStream range1 = IntStream.range(1,5);//前闭后开[1,5)
IntStream range1 = IntStream.rangeClosed(1,5);//前闭后闭[1,5]
//创建一个无限流
//limit(5)是限制流中元素5个
IntStream ints = new Random().ints().limit(5);
//效果一样
IntStream ints = new Random.ints(5);
//自定义流
Stream.generate(() -> Math.random()).limit(5).peek(System.out::println).count();
```

### 并行流和串行流

串行流：由一个线程，串行的方式对流的数据进行处理

并行流：以多个线程，并行的方式对流的数据进行处理

> .parallel()这个是无序的，不用我们自己去开线程
>
> 几个线程可以看一下
>
> .parallel()的位置只要在终止方法前就行
>
> 串并行混合处理
>
> .parallel()
>
> .处理
>
> .sequential()
>
> .处理
>
> 这个意思很明确，串并行执行效果为后者，执行效果按后设置的方式处理（如果设置多个，起作用的只有最后一个）

到底有多少个线程：

> 默认线程数是CPU逻辑处理器的数量
>
> 可以设置线程数：
>
> //修改默认线程池中的线程数量
>
> //默认的使用这个线程池ForkJoinPool.commonPool
>
> String key = "java.util.concurrent.ForkJoinPool.common.parallelism";
>
> System.setProperty(key,"15");//指定默认线程池中的数量为16，其中包含指定的15个加main
>
> 可以指定自定义线程池，不用默认的
>
> //使用自定义线程池
>
> ForkJoinPool pool = new ForkJoinPool(4);
>
> //这里使用一个函数式接口的任务
>
> pool.submit(() -> IntStream.rang(1,100)
> 								.parallel()
> 								.peek(Test::print)
> 								.count());
>
> synchronized(pool){
> 			//wait(),notify(),notifyAll()必须在同步方法或同步代码块中
> 			//且哪个对象调用这些方法，哪个对象就要同步加锁
> 			pool.wait()
> 		}

## Stream流的中间操作

### 无状态操作

对于数据池中的元素，我们也称为流中的元素

#### map(Function<T,R> action)

map是有输入有输出，这是一个无状态操作，当前元素是否应该被过滤和之前的元素数据无关系,把一个值映射成另外一个值

```java
String words = "Beijing welcome you I love china";
Stream.of(words.splite(" "))
    .map(word -> word.length())
    .forEach(System.out::println);
//注意以下代码
String words = "Beijing welcome you I love china";
Stream.of(words.splite(" "))
    .peek(System.out::print)
    .map(word -> word.length())
    .forEach(System.out::println);
输出效果是
    Beijing7
    welcome7
    you3
    ....
 //也就是说，数据池里面的元素的执行顺序，并不是把所有的元素拿出来执行完一个中间方法再执行后面的中间方法
 //而是，把其中一个元素拿出来执行完所有中间方法后，在遍历执行第二个中间方法
 //老雷的描述，学习学习
 //当所有操作都是无状态时
 //流中的元素对于操作的执行并非是将流中所有元素按照顺序先执行完一个操作，再让所有元素执行完第二个操作
 //而是逐个拿出元素，将所有操作执行完毕之后，再拿出一个元素，将所有操作在执行完毕
```

#### mapToXXX

mapToDouble

mapToObject

mapToXXX

#### flatMap

```java
String words = "Beijing welcome you I love china";
Stream.of(words.split(" "))
    .flatMap(word -> word.chars().boxed())//flatMap中的参数为Function,且要求返回类型为Stream
    .forEach(ch -> System.out.println((char)ch.intVale()));
Stream.of(words.split(" "))
    .flatMap(word -> Stream.of(word.split("")))//flatMap中的参数为Function,且要求返回类型为Stream
    .forEach(System.out :: println)
```

#### peek()

peek的效果和forEach差不多，只不过peek()是中间操作

#### filter

```java
String words = "Beijing welcome you I love china";
Stream.of(words.split(" "))
    .filter(word -> word.length > 4)//当过滤条件为true时，当前元素会保留在流中，否则从流中删除
    .forEach(System.out :: println);
```

### 有状态操作

#### distinct()

当前元素是否应该被过滤和之前的元素数据有关系

```java
Stream.of(words.split(" "))
    .flatMap(word -> Stream.of(word.split("")))//flatMap中的参数为Function,且要求返回类型为Stream
    .distinct()//去除重复字符
    .forEach(System.out :: println)
```

#### sorted()

```java
Stream.of(words.split(" "))
    .flatMap(word -> Stream.of(word.split("")))//flatMap中的参数为Function,且要求返回类型为Stream
    .sorted()//默认按照字母序进行排序
    .forEach(System.out :: println);
//增加比较器
Stream.of(words.split(" "))
    .flatMap(word -> Stream.of(word.split("")))//flatMap中的参数为Function,且要求返回类型为Stream
    .sorted((d1,d2) -> d1.charAt(0) - d2.charAt(0))//默认按照字母序进行排序
    .forEach(System.out :: println);
Stream.of(words.split(" "))
    .flatMap(word -> Stream.of(word.split("")))//flatMap中的参数为Function,且要求返回类型为Stream
    .sorted((d1,d2) -> d2.charAt(0) - d1.charAt(0))//逆序
    .forEach(System.out :: println);
```

#### skip

```java
Stream.of(words.split(" "))
    .flatMap(word -> Stream.of(word.split("")))//flatMap中的参数为Function,且要求返回类型为Stream
    .distinct()//去除重复字符
    .sorted()
    .skip(2)//指定跳过n个元素
    .forEach(System.out :: print);
```

#### 特殊的

```java
Stream.of(words.split(" "))
    .flatMap(word -> Stream.of(word.split("")))//flatMap中的参数为Function,且要求返回类型为Stream
    .distinct()//去除重复字符
    .peek(System.out :: println)
    .sorted()
    .forEach(System.err :: println);
//无状态操作里混着有状态操作，那么执行顺序应该是怎样呢？
//以有状态的操作为分水岭
```

## 流的终止操作

终止操作分类：终止操作的结束意味着Stream流操作结束。根据操作是否需要遍历流中的所有元素，可以将终止操作划分为短路操作与非短路操作

### 非短路操作

#### forEachOrdered

```java
String words = "Beijing is the capital of China";
Stream.of(words.split(" "))
    //.forEachOrdered(System.out :: println);//与forEach在串行操作时无区别
    .forEach(System.out :: println);//与forEach在串行操作时无区别
//并行
String words = "Beijing is the capital of China";
Stream.of(words.split(" "))
    .parallel()
    //.forEachOrdered(System.out :: println);//并行时，有序
    .forEach(System.out :: println);//并行时，无序
```

#### collect

##### TODO

#### toArray

收集到数组中

#### count

统计有多少元素

#### reduce

```java
String words = "Beijing is the capital of China";
Optional<Integer> reduce = Stream.of(words.split(" "))
    .map(word -> word.length())
    .reduce((s1,s2) -> s1 + s1);//每两个元素做一个相加，最后变成一个数

System.out.println(reduce.get());

String words = "Beijing is the capital of China";
Optional<Integer> reduce = Stream.of(words.split(" "))
    .map(word -> word.length())
    .filter(n -> n > 200)
    .reduce((s1,s2) -> s1 + s1);//没两个元素做一个相加，最后变成一个数
System.out.println(reduce.orElse(-1));

Integer reduce = Stream.of(words.split(" "))
    .map(word -> word.length())
    .filter(n -> n > 200)
    .reduce(0,(s1,s2) -> s1 + s1);//这样也可以
//但是注意，这里不能放-1，因为如果放-1会影响运算，老铁，高效开发不是这那是什么？
Stream.of(words.split(" "))
    .filter(n -> n.length > 200)
    .reduce(null,(s1,s2) -> s1 + "," + s1);//注意，这种情况会在单词最前面加上一个null,应该用Optional
```

#### max

```java
//找一个单词长度最长的单词,注意的是匹配第一个
Optional<String> optional = Stream.of(words.split(" ")).max((s1,s2) -> s1.length - s2.length);
////找一个单词长度最长的单词，注意匹配的是第一个
Optional<String> optional = Stream.of(words.split(" ")).max((s1,s2) -> s2.length - s1.length);

```

#### min

同上

### 短路操作

```java
//判断所有单词长度是否大于3
Stream.of(words.split(" ")).allMatch(word -> word.length() > 3);//结果为false,只要找到一个不大于3的，就返回了，所以是短路的
//是否存在单词长度大于3的单词
Stream.of(words.split(" ")).anyMatch(word -> word.length() > 3);//结果为true,只要找到一个大于3的，就返回了，所以是短路的
//是否不存在单词长度大于3的单词
Stream.of(words.split(" ")).noneMatch(word -> word.length() > 3);//结果为false,只要找到一个大于3的，就返回了，所以是短路的
Optional<String> optional = Stream.of(words.split(" ")).findFirst();//只要找到第一个就结束查询
Optional<String> optional = Stream.of(words.split(" ")).findAny();//只要找到任意一个就返回
```

## 收集器

收集器的功能非常强大，强大，强大，强

专题介绍

```java
public class Student{}{
    private String name;
    private String school;
    private String gender;
}
private List<Student> students;

@Before
public void before(){
    Student student0 = new Student("周零","","");
    Student student1 = new Student("郑一","","");
    Student student2 = new Student("吴二","","");
    Student student3 = new Student("张三","","");
    Student student4 = new Student("李四","","");
    Student student5 = new Student("王五","","");
    Student student6 = new Student("赵六","","");
    Student student7 = new Student("钱七","","");
    Student student8 = new Student("秦八","","");
    Student student9 = new Student("段九","","");

    students = Arrays.asList();
}
@Test
public void test01(){
    //获取所有学生名单
    students.stream.map(Student :: getName).collect(Collectors.toList());

    students.stream.map(Student :: getSchool).distinct().collect(Collectors.toList());
    students.stream.map(Student :: getSchool).collect(Collectors.toSet());//用的HashSet

    //指定treeSet
    students.stream.map(Student :: getSchool).collect(Collectors.toCollection(TreeSet :: new));

    //返回map
    Map<String, List<Student>> map = students.stream.collect(Collectors.groupingBy(Student :: getSchool));

    //添加appach.commons Collections
    //使用工具类显示map中的key-value
    MapUtils.verbosePrint(System.out ,"学校" , map)


}

//统计各个学校人数
Map<String,Long> schoolCount = students.stream.collect(Collectors.groupingBy(Student :: getSchool.Collections.counting()))


    //分块
    //按照性别分组
    Map<String, List<Student>> map = students.stream.collect(Collectors.groupingBy(Student :: getGender));

Map<Boolean, List<Student>> map = students.stream.collect(Collectors.partitioningBy(s -> "男".equals(s.getGender())));
//返回值得key是：true    false
//分块是分组的一种特殊情况


//按95位标准，分组
Map<Boolean, List<Student>> map = students.stream.collect(Collectors.partitioningBy(s -> s.getScore() > 95)));
//返回值得key是：true    false

//按95分组，计算平均分
Map<Boolean, Double> map = students.stream.collect(Collectors.partitioningBy(s -> s.getScore() > 95,Collectors.averagingDouble(Student :: getScore))));


//
DoubleSummaryStatistics scoreSummary = students.stream.collect(Collectors.summarizingDouble(Student :: getScore);
```

