# 关于java8 parallelStream线程安全的思考

## 背景

java8的stream接口极大的地减少了for循环写法的复杂性，stream提供了map/reduce/collect等一系列聚合接口，还支持并发操作：parallelStream。

在日常开发过程中，经常会遇到遍历一个很大的集合做重复的操作，这个时候如果使用串行执行会相当耗时，因此一般会采用多线程来提速。Java8的parallelStream用fork/join框架提供了并发执行能力。但是如果使用不当，很容易陷入误区。

<font color ='red'>在设备健康度和风险系统中，对大集合的相同处理，我引入了forkJoin的概念，但殊不知自己是多此一举，因为在java8中，有并行流的存在，不需要手工去构建forkjoin框架，这里真正验证了一句话，没文化，真可怕。因为半分钟的事情，我可能花了半天的时间。**值得深思**</font>

## java8的parallelStream是线程安全的吗

一个简单的例子，在下面的代码中，采用stream的forEach接口对1-10000进行遍历，分别插入到3个ArrayList中，其中对第一个list的插入采用串行遍历，第二个使用parallelStream,第三个使用parallelStream的同时用ReentryLock对插入列表操作进行同步。

```java
	@Test
    public void test10(){
        String key = "java.util.concurrent.ForkJoinPool.common.parallelism";
        System.setProperty(key,"64");
        List<Integer> list1 = new ArrayList<>();
        List<Integer> list2 = new ArrayList<>();
        List<Integer> list3 = new ArrayList<>();
        IntStream.rangeClosed(1,100000).forEach(list1 :: add);
        IntStream.rangeClosed(1,100000).parallel().forEach(list2 :: add);
        ReentrantLock lock = new ReentrantLock();
        IntStream.rangeClosed(1,100000).parallel().forEach(p -> {
            lock.lock();
            try {
                list3.add(p);
            }finally {
                lock.unlock();
            }
        });
        System.out.println(list1.size());
        System.out.println(list2.size());
        System.out.println(list3.size());
    }


//执行结果
100000
98389
100000
```

显而易见，stream.parallel().forEach()中执行的操作并非是线程安全的

那么既然parallelStream不是线程安全的，是不是在其中的进行的非原子操作都要加锁呢？

stackOverflow上的答案：

- https://codereview.stackexchange.com/questions/60401/using-java-8-parallel-streams
- https://stackoverflow.com/questions/22350288/parallel-streams-collectors-and-thread-safety

在上面的两个问题的解答中，证实了parallelStream的forEach接口确实不能保证同步，同时也提出了解决方案：用collect和reduce接口。

http://docs.oracle.com/javase/tutorial/collections/streams/parallelism.html

在javadoc中也对stream的并发操作进行相关介绍

Collections框架提供同步的包装，使得其中的操作线程安全。

### stream的collect接口

Stream.java中的collect方法句柄：

```java
<R, A> R collect(Collector<? super T, A, R> collector);
```

在该实现方法中，参数是一个Collector对象，可以使用Collectors类的静态方法构造Collector对象，比如Collectors.toList(),toSet(),toMap(),等等

```java
//自己
list4 = IntStream.rangeClosed(1,1000000).parallel().collect(ArrayList :: new, ArrayList::add, ArrayList::addAll);
//Collectors类的静态方法
list2 = list1.parallelStream().collect(Collectors.toList());
//该静态方法返回的和自己实现的起始一样
public static <T>
    Collector<T, ?, List<T>> toList() {
        return new CollectorImpl<>((Supplier<List<T>>) ArrayList::new, List::add,
                                   (left, right) -> { left.addAll(right); return left; },
                                   CH_ID);
    }
```







 





