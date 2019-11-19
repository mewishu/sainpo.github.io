---
title: Java8新特性
date: 2019-09-05 10:55:01
categories: 
- java
- Java8新特性
tags:
- Java8新特性
---

# 接口的默认方法和静态方法

在Java8之前，接口中**只能**包含抽象方法。那么这有什么样弊端呢？比如，想再Collection接口中添加一个spliterator抽象方法，那么也就意味着之前所有实现Collection接口的实现类，都要重新实现spliterator这个方法才行。而接口的默认方法就是**为了解决接口的修改与接口实现类不兼容的问题，作为代码向前兼容的一个方法**。

那么如何在接口中定义一个默认方法呢？来看下JDK中Collection中如何定义spliterator方法的：

```java
default Spliterator<E> spliterator() {
    return Spliterators.spliterator(this, 0);
}
```

可以看到定义接口的默认方法是通过**default**关键字。因此，在Java8中接口能够包含抽象方法外还能够包含若干个默认方法（即有完整逻辑的实例方法）。

```java
public interface IAnimal {
    default void breath(){
        System.out.println("breath!");
    };
}


public class DefaultMethodTest implements IAnimal {
    public static void main(String[] args) {
        DefaultMethodTest defaultMethod = new DefaultMethodTest();
        defaultMethod.breath();
    }

}


输出结果为：breath!
```

可以看出**IAnimal**接口中有由default定义的默认方法后，那么其实现类DefaultMethodTest也同样能够拥有实例方法**breath**。但是如果一个类继承多个接口，多个接口中有相同的方法就会产生冲突该如何解决？实际上默认方法的改进，使得java类能够拥有类似多继承的能力，即一个对象实例，将拥有多个接口的实例方法，自然而然也会存在方法重复冲突的问题。

下面来看一个例子：

```java
public interface IDonkey{
    default void run() {
        System.out.println("IDonkey run");
    }
}

public interface IHorse {

    default void run(){
        System.out.println("Horse run");
    }

}

public class DefaultMethodTest implements IDonkey,IHorse {
    public static void main(String[] args) {
        DefaultMethodTest defaultMethod = new DefaultMethodTest();
        defaultMethod.breath();
    }



}
```

定义两个接口：IDonkey和IHorse，这两个接口中都有相同的run方法。DefaultMethodTest实现了这两个接口，由于这两个接口有相同的方法，因此就会产生冲突，不知道以哪个接口中的run方法为准，编译会出错：`inherits unrelated defaults for run.....`

> 解决方法

针对由默认方法引起的方法冲突问题，**只有通过重写冲突方法，并方法绑定的方式，指定以哪个接口中的默认方法为准**。

```java
public class DefaultMethodTest implements IAnimal,IDonkey,IHorse {
    public static void main(String[] args) {
        DefaultMethodTest defaultMethod = new DefaultMethodTest();
        defaultMethod.run();
    }

    @Override
    public void run() {
        IHorse.super.run();
    }
}
```

DefaultMethodTest重写了run方法，并通过 `IHorse.super.run();`指定以IHorse中的run方法为准。

**静态方法**

在Java8中还有一个特性就是，接口中还可以声明静态方法，如下例：

```java
public interface IAnimal {
    default void breath(){
        System.out.println("breath!");
    }
    static void run(){}
}
```

# 函数式接口FunctionInterface与lambda表达式

参见: https://sainpo.top/2019/09/03/java8-lambda/

# 方法引用

方法引用是为了进一步简化lambda表达式，通过类**名或者实例名与方法名的组合来直接访问到类或者实例已经存在的方法或者构造方法**。方法引用使用**::**来定义，**::**的前半部分表示类名或者实例名，后半部分表示方法名，如果是构造方法就使用`NEW`来表示。

方法引用在Java8中使用方式相当灵活，总的来说，一共有以下几种形式：

- 静态方法引用：ClassName::methodName;
- 实例上的实例方法引用：instanceName::methodName;
- 超类上的实例方法引用：supper::methodName;
- 类的实例方法引用：ClassName:methodName;
- 构造方法引用Class:new;
- 数组构造方法引用::TypeName[]::new

下面来看一个例子：

```java
public class MethodReferenceTest {

    public static void main(String[] args) {
        ArrayList<Car> cars = new ArrayList<>();
        for (int i = 0; i < 5; i++) {
            Car car = Car.create(Car::new);
            cars.add(car);
        }
        cars.forEach(Car::showCar);

    }

    @FunctionalInterface
    interface Factory<T> {
        T create();
    }

    static class Car {
        public void showCar() {
            System.out.println(this.toString());
        }

        public static Car create(Factory<Car> factory) {
            return factory.create();
        }
    }
}


输出结果：

learn.MethodReferenceTest$Car@769c9116
learn.MethodReferenceTest$Car@6aceb1a5
learn.MethodReferenceTest$Car@2d6d8735
learn.MethodReferenceTest$Car@ba4d54
learn.MethodReferenceTest$Car@12bc6874
```

在上面的例子中使用了`Car::new`，即通过构造方法的方法引用的方式进一步简化了lambda的表达式，`Car::showCar`，即表示实例方法引用。

# Stream

Java8中有一种新的数据处理方式，那就是流Stream，结合lambda表达式能够更加简洁高效的处理数据。Stream使用一种类似于SQL语句从数据库查询数据的直观方式，对数据进行如筛选、排序以及聚合等多种操作。

## 什么是流Stream

Stream是一个来自数据源的元素队列并支持聚合操作，更像是一个更高版本的Iterator,原始版本的Iterator，只能一个个遍历元素并完成相应操作。而使用Stream，只需要指定什么操作，如“过滤长度大于10的字符串”等操作，Stream会内部遍历并完成指定操作。

Stream中的元素在管道中经过中间操作（intermediate operation）的处理后，最后由最终操作（terminal operation）得到最终的结果。

- 数据源：是Stream的来源，可以是集合、数组、I/O channel等转换而成的Stream；
- 基本操作：类似于SQL语句一样的操作，比如filter,map,reduce,find,match,sort等操作。

当我们操作一个流时，实际上会包含这样的执行过程：

**获取数据源-->转换成Stream-->执行操作，返回一个新的Stream-->再以新的Stream继续执行操作--->直至最后操作输出最终结果**。

## 生成Stream的方式

生成Stream的方式主要有这样几种：

1. 从接口Collection中和Arrays：
   - Collection.stream();
   - Collection.parallelStream(); //相较于串行流，并行流能够大大提升执行效率
   - Arrays.stream(T array);
2. Stream中的静态方法：
   - Stream.of()；
   - generate(Supplier s);
   - iterate(T seed, UnaryOperator f);
   - empty();
3. 其他方法
   - Random.ints()
   - BitSet.stream()
   - Pattern.splitAsStream(java.lang.CharSequence)
   - JarFile.stream()
   - BufferedReader.lines()

下面对前面常见的两种方式给出示例：

```java
public class StreamTest {


    public static void main(String[] args) {
        //1.使用Collection中的方法和Arrays
        String[] strArr = new String[]{"a", "b", "c"};
        List<String> list = Arrays.asList(strArr);
        Stream<String> stream = list.stream();
        Stream<String> stream1 = Arrays.stream(strArr);

        //2. 使用Stream中提供的静态方法
        Stream<String> stream2 = Stream.of(strArr);
        Stream<Double> stream3 = Stream.generate(Math::random);
        Stream<Object> stream4 = Stream.empty();
        Stream.iterate(1, i -> i++);

    }
}	
```

## Stream的操作

常见的Stream操作有这样几种：

1. Intermediate（中间操作）:中间操作是指对流中数据元素做出相应转换或操作后依然返回为一个流Stream，仍然可以供下一次流操作使用。常用的有：map (mapToInt, flatMap 等)、 filter、 distinct、 sorted、 peek、 limit、 skip。
2. Termial（结束操作）：是指最终对Stream做出聚合操作，输出结果。

### **中间操作**

#### Filter(过滤)

Filter接收一个判断用来过滤流中的所有元素. 这个操作是中间操作，它能够使我们对结果进行另一个流操作(`forEach`) . ForEach接受一个consumer操作对每一个过滤的流元素中. ForEach是一个终端操作. 它返回值void,所以我们不能调用另一个函数操作.

```java
stringCollection
    .stream()
    .filter((s) -> s.startsWith("a"))
    .forEach(System.out::println);

// "aaa2", "aaa1"
```

#### Map(映射)

对Stream中元素按照指定规则映射成另一个元素，将每一个元素都添加字符串“_map”

```java
stream.map(str -> str + "_map").forEach(System.out::println);
```

map方法是一对一的关系，将stream中的每一个元素按照映射规则成另外一个元素，而如果是一对多的关系的话就需要使用flatmap方法。

#### concat(对流进行合并操作)

concat方法将两个Stream连接在一起，合成一个Stream。若两个输入的Stream都时排序的，则新Stream也是排序的；若输入的Stream中任何一个是并行的，则新的Stream也是并行的；若关闭新的Stream时，原两个输入的Stream都将执行关闭处理。

```java
Stream.concat(Stream.of(1, 2, 3), Stream.of(4, 5, 6)).
	forEach(System.out::println);
```

#### distinct(对流进行去重操作)

去除流中重复的元素

```java
Stream<String> stream = Stream.of("a", "a", "b", "c");
        stream.distinct().forEach(System.out::println);

输出结果：
a
b
c
```

#### limit：限制流中元素的个数

截取流中前两个元素：

```java
Stream<String> stream = Stream.of("a", "a", "b", "c");
        stream.limit(2).forEach(System.out::println);

输出结果：
a
a
```

#### skip：跳过流中前几个元素

丢掉流中前两个元素：

```java
Stream<String> stream = Stream.of("a", "a", "b", "c");
        stream.skip(2).forEach(System.out::println);
输出结果：
b
c
```

#### peek：对流中每一个元素依次进行操作，类似于forEach操作

JDK中给出的例子：

```java
Stream.of("one", "two", "three", "four")
            .filter(e -> e.length() > 3)
            .peek(e -> System.out.println("Filtered value: " + e))
            .map(String::toUpperCase)
            .peek(e -> System.out.println("Mapped value: " + e))
            .collect(Collectors.toList());
输出结果：
Filtered value: three
Mapped value: THREE
Filtered value: four
Mapped value: FOUR
```

#### Sorted(排序)

```java
Stream<Integer> stream = Stream.of(3, 2, 1);
        stream.sorted(Integer::compareTo).forEach(System.out::println);
输出结果：
1
2
3
```

注意 `sorted` 仅仅是创建一个排序后的视图操作，并没有操作排序返回的集合. 排序的 `stringCollection`并没有受到影响。

#### Match(匹配)

Stream 有三个 match 方法，从语义上说：

- allMatch：Stream 中全部元素符合传入的 predicate，返回 true；
- anyMatch：Stream 中只要有一个元素符合传入的 predicate，返回 true；
- noneMatch：Stream 中没有一个元素符合传入的 predicate，返回 true。

如检查Stream中每个元素是否都大于5：

```java
Stream<Integer> stream = Stream.of(3, 2, 1);
boolean match = stream.allMatch(integer -> integer > 5);
System.out.println(match);
输出结果：
false
```

### **结束操作**

#### collector

参见: [【译】java8之collector](https://www.jianshu.com/p/c0d5c3094324)

#### Count(统计)

```java
long count = stream.filter(str -> str.isEmpty()).count();
```

#### max/min(找出流中最大或者最小的元素)

```java
Stream<Integer> stream = Stream.of(3, 2, 1);
    System.out.println(stream.max(Integer::compareTo).get());

输出结果：
3
```

#### forEach(遍历)

forEach方法前面已经用了好多次，其用于遍历Stream中的所元素，避免了使用for循环，让代码更简洁，逻辑更清晰。

示例：

```java
Stream.of(5, 4, 3, 2, 1)
    .sorted()
    .forEach(System.out::println);
    // 打印结果
    // 1，2，3,4,5
```

#### Reduce(合并)

这个终端操作完成一个流中元素合并操作通过给定的函数.返回的结果通过 `Optional来保存值`.

```java
Optional<String> reduced =
    stringCollection
        .stream()
        .sorted()
        .reduce((s1, s2) -> s1 + "#" + s2);

reduced.ifPresent(System.out::println);
// "aaa1#aaa2#bbb1#bbb2#bbb3#ccc#ddd1#ddd2"
```

# Optional

为了解决空指针异常，在Java8之前需要使用if-else这样的语句去防止空指针异常，而在Java8就可以使用Optional来解决。Optional可以理解成一个数据容器，甚至可以封装null，并且如果值存在调用isPresent()方法会返回true。为了能够理解Optional。先来看一个例子：

```java
public class OptionalTest {


    private String getUserName(User user) {
        return user.getUserName();
    }

    class User {
        private String userName;

        public User(String userName) {
            this.userName = userName;
        }

        public String getUserName() {
            return userName;
        }
    }
}
```

事实上，getUserName方法对输入参数并没有进行判断是否为null，因此，该方法是不安全的。如果在Java8之前，要避免可能存在的空指针异常的话就需要使用`if-else`进行逻辑处理，getUserName会改变如下：

```java
private String getUserName(User user) {
    if (user != null) {
        return user.getUserName();
    }
    return null;
}
```

这是十分繁琐的一段代码。而如果使用Optional则会要精简很多：

```java
private String getUserName(User user) {
    Optional<User> userOptional = Optional.ofNullable(user);
    return userOptional.map(User::getUserName).orElse(null);
}
```

Java8之前的if-else的逻辑判断，这是一种命令式编程的方式，而使用Optional更像是一种函数式编程，关注于最后的结果，而中间的处理过程交给JDK内部实现。

## Optional.of()或者Optional.ofNullable()

创建Optional对象，差别在于of不允许参数是null，而ofNullable则无限制。

```java
// 参数不能是null
Optional<Integer> optional1 = Optional.of(1);

// 参数可以是null
Optional<Integer> optional2 = Optional.ofNullable(null);

// 参数可以是非null
Optional<Integer> optional3 = Optional.ofNullable(2);
```

## Optional.empty()：所有null包装成的Optional对象：

```java
Optional<Integer> optional1 = Optional.ofNullable(null);
Optional<Integer> optional2 = Optional.ofNullable(null);
System.out.println(optional1 == optional2);// true
System.out.println(optional1 == Optional.<Integer>empty());// true

Object o1 = Optional.<Integer>empty();
Object o2 = Optional.<String>empty();
System.out.println(o1 == o2);// true
```



## isPresent()：判断值是否存在

```java
Optional<Integer> optional1 = Optional.ofNullable(1);
Optional<Integer> optional2 = Optional.ofNullable(null);

// isPresent判断值是否存在
System.out.println(optional1.isPresent() == true);
System.out.println(optional2.isPresent() == false);
```

## ifPresent(Consumer consumer)

如果option对象保存的值不是null，则调用consumer对象，否则不调用

```java
Optional<Integer> optional1 = Optional.ofNullable(1);
Optional<Integer> optional2 = Optional.ofNullable(null);

// 如果不是null,调用Consumer
optional1.ifPresent(new Consumer<Integer>() {
	@Override
	public void accept(Integer t) {
		System.out.println("value is " + t);
	}
});

// null,不调用Consumer
optional2.ifPresent(new Consumer<Integer>() {
	@Override
	public void accept(Integer t) {
		System.out.println("value is " + t);
	}
});


```

## orElse(value)

如果optional对象保存的值不是null，则返回原来的值，否则返回value

```java
Optional<Integer> optional1 = Optional.ofNullable(1);
Optional<Integer> optional2 = Optional.ofNullable(null);

// orElse
System.out.println(optional1.orElse(1000) == 1);// true
System.out.println(optional2.orElse(1000) == 1000);// true
```



## orElseGet(Supplier supplier)

功能与orElse一样，只不过orElseGet参数是一个对象

```java
Optional<Integer> optional1 = Optional.ofNullable(1);
Optional<Integer> optional2 = Optional.ofNullable(null);

System.out.println(optional1.orElseGet(() -> {
	return 1000;
}) == 1);//true

System.out.println(optional2.orElseGet(() -> {
	return 1000;
}) == 1000);//true
```

## orElseThrow()

值不存在则抛出异常，存在则什么不做，有点类似Guava的Precoditions

```java
Optional<Integer> optional1 = Optional.ofNullable(1);
Optional<Integer> optional2 = Optional.ofNullable(null);

optional1.orElseThrow(()->{throw new IllegalStateException();});

try
{
	// 抛出异常
	optional2.orElseThrow(()->{throw new IllegalStateException();});
}
catch(IllegalStateException e )
{
	e.printStackTrace();
}
```

## filter(Predicate)

判断Optional对象中保存的值是否满足Predicate，并返回新的Optional。

```java
Optional<Integer> optional1 = Optional.ofNullable(1);
Optional<Integer> optional2 = Optional.ofNullable(null);

Optional<Integer> filter1 = optional1.filter((a) -> a == null);
Optional<Integer> filter2 = optional1.filter((a) -> a == 1);
Optional<Integer> filter3 = optional2.filter((a) -> a == null);
System.out.println(filter1.isPresent());// false
System.out.println(filter2.isPresent());// true
System.out.println(filter2.get().intValue() == 1);// true
System.out.println(filter3.isPresent());// false
```



## map(Function)

对Optional中保存的值进行函数运算，并返回新的Optional(可以是任何类型)

```java
Optional<Integer> optional1 = Optional.ofNullable(1);
Optional<Integer> optional2 = Optional.ofNullable(null);

Optional<String> str1Optional = optional1.map((a) -> "key" + a);
Optional<String> str2Optional = optional2.map((a) -> "key" + a);

System.out.println(str1Optional.get());// key1
System.out.println(str2Optional.isPresent());// false
```

## flatMap()

功能与map()相似，差别请看如下代码。flatMap方法与map方法类似，区别在于mapping函数的返回值不同。map方法的mapping函数返回值可以是任何类型T，而flatMap方法的mapping函数必须是Optional。

```java
Optional<Integer> optional1 = Optional.ofNullable(1);

Optional<Optional<String>> str1Optional = optional1.map((a) -> {
	return Optional.<String>of("key" + a);
});

Optional<String> str2Optional = optional1.flatMap((a) -> {
	return Optional.<String>of("key" + a);
});

System.out.println(str1Optional.get().get());// key1
System.out.println(str2Optional.get());// key1
```

# Date/time API的改进

在Java8之前的版本中，日期时间API存在很多的问题，比如：

- 线程安全问题：java.util.Date是非线程安全的，所有的日期类都是可变的；
- 设计很差：在java.util和java.sql的包中都有日期类，此外，用于格式化和解析的类在java.text包中也有定义。而每个包将其合并在一起，也是不合理的；
- 时区处理麻烦：日期类不提供国际化，没有时区支持，因此Java中引入了java.util.Calendar和Java.util.TimeZone类；

针对这些问题，Java8重新设计了日期时间相关的API，Java 8通过发布新的Date-Time API (JSR 310)来进一步加强对日期与时间的处理。在java.util.time包中常用的几个类有：

- 它通过指定一个时区，然后就可以获取到当前的时刻，日期与时间。Clock可以替换System.currentTimeMillis()与TimeZone.getDefault()
- Instant:一个instant对象表示时间轴上的一个时间点，Instant.now()方法会返回当前的瞬时点（格林威治时间）；
- Duration:用于表示两个瞬时点相差的时间量；
- LocalDate:一个带有年份，月份和天数的日期，可以使用静态方法now或者of方法进行创建；
- LocalTime:表示一天中的某个时间，同样可以使用now和of进行创建；
- LocalDateTime：兼有日期和时间；
- ZonedDateTime：通过设置时间的id来创建一个带时区的时间；
- DateTimeFormatter：日期格式化类，提供了多种预定义的标准格式；

示例代码如下：

```java
public class TimeTest {
    public static void main(String[] args) {
        Clock clock = Clock.systemUTC();
        Instant instant = clock.instant();
        System.out.println(instant.toString());

        LocalDate localDate = LocalDate.now();
        System.out.println(localDate.toString());

        LocalTime localTime = LocalTime.now();
        System.out.println(localTime.toString());

        LocalDateTime localDateTime = LocalDateTime.now();
        System.out.println(localDateTime.toString());

        ZonedDateTime zonedDateTime = ZonedDateTime.now(ZoneId.of("Asia/Shanghai"));
        System.out.println(zonedDateTime.toString());
    }
}
输出结果为：
2018-04-14T12:50:27.437Z
2018-04-14
20:50:27.646
2018-04-14T20:50:27.646
2018-04-14T20:50:27.647+08:00[Asia/Shanghai]
```

# 其他改进

Java8还在其他细节上也做出了改变，归纳如下：

1. 之前的版本，注解在同一个位置只能声明一次，而Java8版本中提供@Repeatable注解，来实现可重复注解；
2. String类中提供了join方法来完成字符串的拼接；
3. 在Arrays上提供了并行化处理数组的方式，比如利用Arrays类中的parallelSort可完成并行排序；
4. 在Java8中在并发应用层面上也是下足了功夫：（1）提供功能更强大的Future：CompletableFuture；（2）StampedLock可用来替代ReadWriteLock；（3）性能更优的原子类：：LongAdder,LongAccumulator以及DoubleAdder和DoubleAccumulator；
5. 编译器新增一些特性以及提供一些新的Java工具

# 参考资料

Stream的参考资料：

[【译】java8之collector](https://www.jianshu.com/p/c0d5c3094324)



Optional的参考资料：

[JDK8新特性：使用Optional避免null导致的NullPointerException](https://blog.csdn.net/aitangyong/article/details/54564100)