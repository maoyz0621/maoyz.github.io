# JAVA8 之Lambda表达式、函数式接口

## Lambda表达式

Lambda表达式可以看成是匿名内部类（Anonymous Classes）的语法糖，JVM内部通过invokedynamic指令来实现的

### Lambda表达式与匿名内部类的差别

1. 是否生成新类

   

   一、匿名内部类仍然是一个类，编译器会自动为该类取名。

   ```java
   public class MainAnonymousClassTest {
   
       public static void main(String[] args) {
           new Thread(new Runnable() {
               @Override
               public void run() {
                   System.out.println("匿名内部类： ");
               }
           }).start();
       }
   }
   ```

   

   编译之后，生成2个class文件，MainAnonymousClassTest$1.class（匿名内部类产生的）、MainAnonymousClassTest.class

   

   ```java
   $ javap -c -p MainAnonymousClassTest.class
   Compiled from "MainAnonymousClassTest.java"
   public class com.myz.java.study.java8.lambda.MainAnonymousClassTest {
     public com.myz.java.study.java8.lambda.MainAnonymousClassTest();
       Code:
          0: aload_0
          1: invokespecial #1                  // Method java/lang/Object."<init>":()V
          4: return
   
     public static void main(java.lang.String[]);
       Code:
          0: new           #2                  // class java/lang/Thread 为对象分配内存空间，并将地址压入操作数栈顶
          3: dup								// 复制操作数栈顶，压入栈底(此时操作数栈上有连续相同的两个对象地址)
          4: new           #3                  // class com/myz/java/study/java8/lambda/MainAnonymousClassTest$1 /* 创建内部类 */
          7: dup
          8: invokespecial #4                  // Method com/myz/java/study/java8/lambda/MainAnonymousClassTest$1."<init>":()V
         11: invokespecial #5                  // Method java/lang/Thread."<init>":(Ljava/lang/Runnable;)V
         14: invokevirtual #6                  // Method java/lang/Thread.start:()V
         17: return							// 指令结束
   }
   ```

   补充：javap 命令

   ```kotlin
    -l                       输出行号和本地变量表
    -public                  仅显示公共类和成员
    -protected               显示受保护的/公共类和成员
    -package                 显示显示程序包/受保护的/公共类 和成员 (默认)
    -p  -private             显示所有类和成员
    -c                       对代码进行反汇编
    -s                       输出内部类型签名
    -sysinfo                 显示正在处理的类的系统信息 (路径, 大小, 日期, MD5 散列)
    -constants               显示静态最终常量
    -classpath <path>        指定查找用户类文件的位置
    -bootclasspath <path>    覆盖引导类文件的位置
   ```

   二、Lambda表达式不会产生新的类。

   

   ```java
   public class MainLambdaTest {
   
       public static void main(String[] args) {
           new Thread(() -> {
               System.out.println("Lambda实现Runnable");
   
           }).start();
       }
   }
   ```

   

   编译之后，只生成1个class文件.MainLambdaTest.class，**Lambda表达式被封装成主类的私有方法，之后通过invokedynamic指令调用**

   ```java
   $ javap -c -p MainLambdaTest.class
   Compiled from "MainLambdaTest.java"
   public class com.myz.java.study.java8.lambda.MainLambdaTest {
     public com.myz.java.study.java8.lambda.MainLambdaTest();
       Code:
          0: aload_0
          1: invokespecial #1                  // Method java/lang/Object."<init>":()V
          4: return
   
     public static void main(java.lang.String[]);
       Code:
          0: new           #2                  // class java/lang/Thread
          3: dup
          4: invokedynamic #3,  0              // InvokeDynamic #0:run:()Ljava/lang/Runnable; /*使用invokedynamic指令执行*/
          9: invokespecial #4                  // Method java/lang/Thread."<init>":(Ljava/lang/Runnable;)V
         12: invokevirtual #5                  // Method java/lang/Thread.start:()V
         15: return
   
     private static void lambda$main$0();	    /*Lambda表达式被封装成主类的私有方法，之后通过invokedynamic指令调用*/
       Code:
          0: getstatic     #6                  // Field java/lang/System.out:Ljava/io/PrintStream;
          3: ldc           #7                  // String Lambda实现Runnable
          5: invokevirtual #8                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
          8: return
   }
   ```

   

1. this指向

   对于匿名类，关键词**this**解读为匿名类本身，而对于 Lambda 表达式，关键词**this**为 Lambda 的外部类。

   ```java
   public void testRunnable() {
       // 对象本身Xxxx@7d6f77cc
       System.out.println(this);
       new Thread(() -> {
           // Xxxx@7d6f77cc，同外部对象一致
           System.out.println(this);
       }).start();
   
       new Thread(new Runnable() {
           @Override
           public void run() {
               // 匿名类自身，Xxxx@28d3f278
               System.out.println(" 匿名类： " + this);
           }
       }).start();
   }
   ```

   

## 函数接口

有且仅有一个抽象方法的接口即可，然后在加上@FunctionalInterface 注解

```java
@FunctionalInterface
public interface ConsumerInterface<T> {

    void accept(T t);

    /**
     * default方法也可以
     */
    default void a() {
    }
    
}
```



## JUF(java.util.function)

### Function< T, R >

函数型接口，接收T对象，返回R对象

```java
	// 接受输入参数，对输入执行所需操作后  返回一个结果。
    R apply(T t);

    // 返回一个 先执行before函数对象apply方法，再执行当前函数对象apply方法的 函数对象。
    default <V> Function<V, R> compose(Function<? super V, ? extends T> before) {
       Objects.requireNonNull(before);
       return (V v) -> apply(before.apply(v));
    }

    // 返回一个 先执行当前函数对象apply方法， 再执行after函数对象apply方法的 函数对象。
    default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
        Objects.requireNonNull(after);
        return (T t) -> after.apply(apply(t));
    }   

    // 返回一个执行了apply()方法之后只会返回输入参数的函数对象。
    static <T> Function<T, T> identity() {
        return t -> t;
    } 
```

测试用例：

```java
public static void main(String[] args) {
    Function<String, String> function = s -> s + " hello";
    String apply = function.apply("abc ->");
    // abc -> hello
    System.out.println(apply);


    Function<String, String> other = s -> s + " !!!";
    String apply1 = function.andThen(other).apply("aaaa");
    // aaaa hello !!!
    System.out.println(apply1);

    String apply2 = function.compose(other).apply("bbbb");
    // bbbb !!! hello
    System.out.println(apply2);

    Function<String, String> f1 = Function.identity();
    System.out.println(f1.apply("hahahah"));

}
```

处理两个参数函数型接口**BiFunction<T, U, R>**，接收T，U对象，返回R对象

```java
    R apply(T t, U u);

    default <V> BiFunction<T, U, V> andThen(Function<? super R, ? extends V> after) {
        Objects.requireNonNull(after);
        return (T t, U u) -> after.apply(apply(t, u));
    }
```



### Consumer< T >

消费型接口，接收T对象，不返回值

```java
    // 接受输入参数，对其执行所需操作
    void accept(T t);

	// 追加
    default Consumer<T> andThen(Consumer<? super T> after) {
        Objects.requireNonNull(after);
        return (T t) -> { accept(t); after.accept(t); };
    }
```

测试用例：

```java
public static void main(String[] args) {
    Map<String, Object> map = Maps.newHashMap();
    Consumer<Map<String, Object>> consumer = param -> {
        param.put("key1", "value1");
    };

    consumer.accept(map);
    // {key1=value1}
    System.out.println(map);

    Consumer<Map<String, Object>> other = param -> {
        param.put("key2", "value2");
    };

    consumer.andThen(other).accept(map);
	// {key1=value1, key2=value2}
    System.out.println(map);
}
```

消费型接口**BiConsumer<T, U>**，接收T，U对象，不返回值

```java
    void accept(T t, U u);

    default BiConsumer<T, U> andThen(BiConsumer<? super T, ? super U> after) {
        Objects.requireNonNull(after);

        return (l, r) -> {
            accept(l, r);
            after.accept(l, r);
        };
    }
```





### Predicate< T >

断言型接口，接收T对象并返回boolean

```java
    
    boolean test(T t);

	// 把传入的函数和当前函数拼接为一个新的函数，必须同时满足两个函数才会返回True
    default Predicate<T> and(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) && other.test(t);
    }

   
    default Predicate<T> negate() {
        return (t) -> !test(t);
    }

	// 把传入的函数和当前函数拼接为一个新的函数，满足其中一个函数就会返回True
    default Predicate<T> or(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) || other.test(t);
    }

   
    static <T> Predicate<T> isEqual(Object targetRef) {
        return (null == targetRef)
                ? Objects::isNull
                : object -> targetRef.equals(object);
    }
```

测试用例：

```java
	/**
     * test() 核心方法
     */
    @Test
    public void testTest() {
        Predicate<Integer> predicate = t -> t > 20;
        // false
        System.out.println(predicate.test(10));
        // false
        System.out.println(predicate.test(20));
        // true
        System.out.println(predicate.test(30));
    }

    /**
     * negate() 对test()方法取非运算
     */
    @Test
    public void testNegate() {
        Predicate<Integer> predicate = t -> t > 20;
        // true
        System.out.println(predicate.negate().test(10));
        // true
        System.out.println(predicate.negate().test(20));
        // false
        System.out.println(predicate.negate().test(30));
    }

    /**
     * and() 把传入的函数和当前函数拼接为一个新的函数，必须同时满足两个函数才会返回True
     */
    @Test
    public void testAnd() {
        Predicate<Integer> predicate1 = t -> t > 20;
        Predicate<Integer> predicate2 = t -> t % 2 == 0;
        Predicate<Integer> predicate = predicate1.and(predicate2);
        // false
        System.out.println(predicate.test(10));
        // true
        System.out.println(predicate.test(30));
        // false
        System.out.println(predicate.test(31));
    }


    /**
     * or() 把传入的函数和当前函数拼接为一个新的函数，满足其中一个函数就会返回True
     */
    @Test
    public void testOr() {
        Predicate<Integer> predicate1 = t -> t > 20;
        Predicate<Integer> predicate2 = t -> t % 2 == 0;
        Predicate<Integer> predicate = predicate1.or(predicate2);
        // true
        System.out.println(predicate.test(10));
        // true
        System.out.println(predicate.test(30));
        // true
        System.out.println(predicate.test(31));
    }

    @Test
    public void testIsEqual() {
        Predicate<String> predicate = Predicate.isEqual("a");
        System.out.println(predicate.test("ab"));
    }
```

断言型接口**BiPredicate<T, U>**，接收T，U对象并返回boolean

```java
    boolean test(T t, U u);

    default BiPredicate<T, U> and(BiPredicate<? super T, ? super U> other) {
        Objects.requireNonNull(other);
        return (T t, U u) -> test(t, u) && other.test(t, u);
    }

    default BiPredicate<T, U> negate() {
        return (T t, U u) -> !test(t, u);
    }
   
    default BiPredicate<T, U> or(BiPredicate<? super T, ? super U> other) {
        Objects.requireNonNull(other);
        return (T t, U u) -> test(t, u) || other.test(t, u);
    }
```



### Supplier< T >

供给型接口，提供T对象（例如工厂），不接收值，返回值

```java
	T get()
```

测试用例

```java
public static void main(String[] args) throws ParseException {
    Supplier<SimpleDateFormat> normalDateFormat = () -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    // 每次调用get()方法时都会调用构造方法创建一个新对象。对象是否同一个hashcode，取决于对象本身
    SimpleDateFormat sdf = normalDateFormat.get();
    // java.text.SimpleDateFormat@4f76f1a0
    System.out.println(sdf);
    Date date = sdf.parse("2019-03-29 23:24:05");
    System.out.println(date);

    SimpleDateFormat other = normalDateFormat.get();
    // java.text.SimpleDateFormat@4f76f1a0
    System.out.println(other);

    Supplier<A> a = A::new;
    // @277050dc
    System.out.println(a.get());
    // @5c29bfd
    System.out.println(a.get());
}

static class A {
    private String name;

    public A() {
    }

    public A(String name) {
        this.name = name;
    }
}
```



## 新增函数接口

接口 | 描述 | 备注
---|---|---
Iterable	| 	|void forEach(Consumer<? super T> action);<br/>Spliterator<T> spliterator()
Collection |  | removeIf(Predicate<? super E> filter);<br/>Stream<E> stream();<br/>Stream<E> parallelStream() 
List	|| sort(Comparator<? super E> c);                               
Map |		|V getOrDefault(Object key, V defaultValue);<br/>void forEach(BiConsumer<? super K, ? super V> action);<br/>void replaceAll(BiFunction<? super K, ? super V, ? extends V> function);<br/>V putIfAbsent(K key, V value);<br/>boolean remove(Object key, Object value);<br/>boolean replace(K key, V oldValue, V newValue);<br/>V replace(K key, V value);<br/>V computeIfAbsent(K key, Function<? super K, ? extends V> mappingFunction);<br/>V computeIfPresent(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction);<br/>V compute(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction);<br/>V merge(K key, V value, BiFunction<? super V, ? super V, ? extends V> remappingFunction);

### Iterable

#### forEach()



```
default void forEach(Consumer<? super T> action) {
    Objects.requireNonNull(action);
    for (T t : this) {
        action.accept(t);
    }
}
```

### Collection

#### spliterator()

返回容器的可拆分迭代器**Spliterator**，可以多次调用trySplit()分成多个迭代器，且元素没有重叠。

```
default Spliterator<E> spliterator() {
    return Spliterators.spliterator(this, 0);
}
```



#### removeIf()

删除容器中所有满足filter指定条件的元素

```java
default boolean removeIf(Predicate<? super E> filter) {
    Objects.requireNonNull(filter);
    boolean removed = false;
    // 元素迭代
    final Iterator<E> each = iterator();
    while (each.hasNext()) {
        if (filter.test(each.next())) {
            each.remove();
            removed = true;
        }
    }
    return removed;
}
```



#### stream() 和 parallelStream()

分别返回该容器的`Stream`视图表示，不同之处在于`parallelStream()`返回并行的`Stream`。

```java
    default Stream<E> stream() {
        return StreamSupport.stream(spliterator(), false);
    }

    default Stream<E> parallelStream() {
        return StreamSupport.stream(spliterator(), true);
    }
```

### List

#### sort()

根据指定的比较规则对容器元素进行排序

```java
    default void sort(Comparator<? super E> c) {
        Object[] a = this.toArray();
        Arrays.sort(a, (Comparator) c);
        ListIterator<E> i = this.listIterator();
        for (Object e : a) {
            i.next();
            i.set((E) e);
        }
    }
```



### Map

#### getOrDefault()

按照给定的key查询`Map`中对应的value，如果没有找到则返回defaultValue

```java
default V getOrDefault(Object key, V defaultValue) {
    V v;
    return (((v = get(key)) != null) || containsKey(key)) ? v : defaultValue;
}
```



#### forEach()

遍历Map值

```java
default void forEach(BiConsumer<? super K, ? super V> action) {
    Objects.requireNonNull(action);
    for (Map.Entry<K, V> entry : entrySet()) {
        K k;
        V v;
        try {
            k = entry.getKey();
            v = entry.getValue();
        } catch(IllegalStateException ise) {
            // this usually means the entry is no longer in the map.
            throw new ConcurrentModificationException(ise);
        }
        action.accept(k, v);
    }
}
```



#### replace() 和 replaceAll()

- `V replace(K key, V value)`，只有在当前`Map`中**`key`的映射存在时**才用`value`去替换原来的值，否则什么也不做．
- `boolean replace(K key, V oldValue, V newValue)`，只有在当前`Map`中**`key`的映射存在且等于`oldValue`时**才用`newValue`去替换原来的值，否则什么也不做．
- void replaceAll(BiFunction<? super K, ? super V, ? extends V> function)，对`Map`中的每个映射执行`function`指定的操作，并用`function`的执行结果替换原来的`value`

```java
default V replace(K key, V value) {
    V curValue;
    if (((curValue = get(key)) != null) || containsKey(key)) {
        curValue = put(key, value);
    }
    return curValue;
}

default boolean replace(K key, V oldValue, V newValue) {
    Object curValue = get(key);
    if (!Objects.equals(curValue, oldValue) || (curValue == null && !containsKey(key))) {
        return false;
    }
    put(key, newValue);
    return true;
}

default void replaceAll(BiFunction<? super K, ? super V, ? extends V> function) {
    Objects.requireNonNull(function);
    for (Map.Entry<K, V> entry : entrySet()) {
        K k;
        V v;
        try {
            k = entry.getKey();
            v = entry.getValue();
        } catch(IllegalStateException ise) {
            // this usually means the entry is no longer in the map.
            throw new ConcurrentModificationException(ise);
        }

        // ise thrown from function is not a cme.
        v = function.apply(k, v);

        try {
            entry.setValue(v);
        } catch(IllegalStateException ise) {
            // this usually means the entry is no longer in the map.
            throw new ConcurrentModificationException(ise);
        }
    }
}
```



#### putIfAbsent()

只有在**不存在key值的映射或映射值为null时**，才将value指定的值放入到`Map`中，否则不对`Map`做更改。

```
default V putIfAbsent(K key, V value) {
	V v = get(key);
	if (v == null) {
		v = put(key, value);
	}

	return v;
}
```



#### remove()

只有在当前`Map`中key正好映射到value时才删除该映射。

```
default boolean remove(Object key, Object value) {
	Object curValue = get(key);
	if (!Objects.equals(curValue, value) || (curValue == null && !containsKey(key))) {
		return false;
	}
	remove(key);
	return true;
}
```



#### merge()

1. 如果`Map`中`key`对应的映射不存在或者为`null`，则将`value`（不能是`null`）关联到`key`上；
2. 否则执行`remappingFunction`，如果执行结果非`null`则用该结果跟`key`关联，否则在`Map`中删除`key`的映射

```
    Map<String, Object> param = new HashMap<>(4);
    param.put("a", 1);
    param.put("b", 2);

    // v1 -> 原本存在的非null值， v2 -> 期望merge的值
    param.merge("a", "a", (v1, v2) -> v1 + ":" + v2);
    // remappingFunction执行结果 null, 则删除key
    param.merge("b", "a", (v1, v2) -> null);
    // 对应的映射不存在或者为null
    param.merge("c", "a", (v1, v2) -> v1 + ":" + v2);

    // {a=1:a, c=a}
    System.out.println(param);
```



#### compute()、computeIfAbsent()、computeIfPresent()

compute：把`remappingFunction`的计算结果关联到`key`上，如果计算结果为`null`，则在`Map`中删除`key`的映射；

computeIfAbsent：只有在当前`Map`中**不存在`key`值的映射或映射值为`null`时**，才调用`mappingFunction`，并在`mappingFunction`执行结果非`null`时，将结果跟`key`关联；

computeIfPresent：只有在当前`Map`中**存在`key`值的映射且非`null`时**，才调用`remappingFunction`，如果`remappingFunction`执行结果为`null`，则删除`key`的映射，否则使用该结果替换`key`原来的映射。



## Stream

1、不是某种数据结构，只是数据源的一种视图；

2、惰性执行，只有等到用户真正需要结果的时候才会执行；

3、可消费性，只能被消费一次，一旦遍历过就会失效，就像容器的迭代器那样，想要再次遍历必须重新生成。

Collection.stream()

Collection.parallelStream()

Arrays.stream(T[] array)

接口继承关系：

```
BaseStream
    IntStream extends BaseStream<Integer, IntStream> 
    LongStream extends BaseStream<Long, LongStream>
    DoubleStream extends BaseStream<Double, DoubleStream>
    Stream<T> extends BaseStream<T, Stream<T>>
```

操作分类：

|    |           |      |
| -- | -- | -- |
|中间操作| 无状态(Stateless)          | unordered() <br/>filter() <br/>map()   对所有元素进行转换，转换前后元素个数不会改变<br>mapToInt() <br/>mapToLong() <br/>mapToDouble() <br/>flatMap()   摊平<br/>flatMapToInt() <br/>flatMapToLong() <br/>flatMapToDouble() <br/>peek() |
|中间操作| 有状态(Stateful)           | distinct()  去除重复元素 <br/>sorted()   排序<br/>limit() <br/>skip() |
|结束操作| 非短路操作                 | forEach() <br/>forEachOrdered() <br/>toArray() <br/>reduce() <br/>collect() <br/>max() <br/>min() <br/>count() |
|结束操作| 短路操作(short-circuiting) | anyMatch() <br>allMatch() <br>noneMatch() <br>findFirst() <br>findAny()</br> |


### 中间操作

惰式执行，调用中间操作只会生成一个标记了该操作的新*stream*。



```java
    /**
     * Intermediate（中间操作）
     * Stream可以进行多次的Intermediate操作，如前面开头的那个例子，其中filter、map、sorted都是Intermediate操作，注意该操作是惰性化的，当调用到该方法的时候，
     * 并没有真正开始Stream的遍历。
     */
    @Test
    public void testMap() {
        List<String> list = Arrays.asList("a1", "a2", "c1", "d1");
        // map -> 新的Stream
        list.stream().map(String::toUpperCase).sorted().forEach(System.out::println);
        // 原数据不变
        System.out.println(list);
    }

    /**
     * flat 扁平的
     */
    @Test
    public void testFlatMap() {
        List<Integer> collect0 = Stream.of(1, 22, 33).flatMap(v -> Stream.of(v * v)).collect(Collectors.toList());
        System.out.println(collect0);

        List<Integer> collect1 = Stream.of(1, 22, 33).map(v -> v * v).collect(Collectors.toList());
        System.out.println(collect1);

        //flatMap的扁平化处理

        List<Map<String, String>> list = new ArrayList<>();
        Map<String, String> map1 = new HashMap<>();
        map1.put("1", "one");
        map1.put("2", "two");

        Map<String, String> map2 = new HashMap<>();
        map2.put("3", "three");
        map2.put("4", "four");
        list.add(map1);
        list.add(map2);

        // 收集map中的val
        Set<String> output = list.stream()
                // 收集val
                .map(Map::values)
                // val 转 String
                .flatMap(Collection::stream)
                // 转 Set
                .collect(Collectors.toSet());
        // [four, one, two, three]
        System.out.println(output);

        // 收集map中的key
        Set<String> collect = list.stream()
                .map(Map::keySet)
                .flatMap(Collection::stream)
                .collect(Collectors.toSet());
        // [1, 2, 3, 4]
        System.out.println(collect);
    }

    /**
     * peek也是对流中的每一个元素进行操作，除了生成一个包含原所有元素的新Stream，还提供一个Consumer消费函数。
     * peek()可以做一些输出、外部处理、副作用等无返回值。生成一个包含原Stream的所有元素的新Stream，新Stream每个元素在被消费之前都会执行peek给定的消费函数;
     */
    @Test
    public void testPeek() {
        List<Integer> list = new ArrayList<>();
        List<Integer> result = Stream.of(1, 2, 3, 4)
                .peek(list::add)
                .map(x -> x * 2)
                .collect(Collectors.toList());
        System.out.println(result);
        System.out.println(list);
    }

    @Test
    public void testFilter() {
        List<String> list = Arrays.asList("a1", "a2", "c1", "d1");
        String[] as = list.stream()
                .filter((s) -> s.startsWith("a"))
                .map(String::toUpperCase)
                .toArray(String[]::new);
        System.out.println(as.length);
    }

    /**
     * 去重
     */
    @Test
    public void testDistinct() {
        List<Integer> collect = Stream.of(1, 2, 3, 3, 3, 2, 4, 5, 6)
                .distinct()
                .collect(Collectors.toList());
        System.out.println(collect);
    }
```


### 结束操作

会触发实际计算，计算发生时会把所有中间操作积攒的操作以*pipeline*的方式执行，这样可以减少迭代次数。计算完成之后*stream*就会失效。

规约操作：

```
java.util.stream.Collector	一个收集函数的接口，声明一个收集器功能
java.util.stream.Collectors	一收集器的工具类，内置了一系列常用收集器的实现，如Collectors.toList()/toSet()，collect()参数
```



**collect()** 

|                                                              |
| :----------------------------------------------------------: |
| ![image-20200319153223436](D:%5CWork%5CIDEA%5Cmaoyz.github.io%5Cimage-20200319153223436.png) |



```
<R, A> R collect(Collector<? super T, A, R> collector)
<R> R collect(Supplier<R> supplier, BiConsumer<R, ? super T> accumulator, BiConsumer<R, R> combiner)
-- supplier    结果存放容器
-- accumulator 累加器执行累加的具体实现
-- combiner    多个容器的聚合策略
-- Function<A, R> finisher()  对结果容器应用最终转换finisher
```



```java
	public void testCollect() {
        List<String> list = Arrays.asList("apple", "banana", "orange", "blueberry", "blackberry");
        List<String> b = list.stream()
                .filter(s -> s.startsWith("b"))
                .collect(Collectors.toList());
        System.out.println(b);
        
        // 使用Collectors.toCollection() 指定想要的
        ArrayList<String> collect = list.stream().collect(Collectors.toCollection(ArrayList::new));

        // 转map
        Map<String, String> map = list.stream()
                .filter(s -> s.startsWith("b"))
                .collect(Collectors.toMap(k -> k, k -> k + "-val"));
        System.out.println(map);

        // 基本类型也可以实现转换（依赖boxed的装箱操作）
        int[] ints = {1, 2, 3};
        List myList = Arrays.stream(ints).boxed().collect(Collectors.toList());
        System.out.println(myList);
    }
```



**reduce()** 

```
Optional<T> reduce(BinaryOperator<T> accumulator)
T reduce(T identity, BinaryOperator<T> accumulator)
<U> U reduce(U identity,BiFunction<U, ? super T, U> accumulator,BinaryOperator<U> combiner)

-- identity: 一个初始化的值；这个初始化的值其类型是泛型U，与Reduce方法返回的类型一致；注意此时Stream中元素的类型是T，与U可以不一样也可以一样，这样的话操作空间就大了；
不管Stream中存储的元素是什么类型，U都可以是任何类型，如U可以是一些基本数据类型的包装类型Integer、Long等；或者是String，又或者是一些集合类型ArrayList等
如果缺少该参数，则没有默认值或初始值，并且它返回 optional
-- accumulator: （累加器）其类型是BiFunction，输入是U与T两个类型的数据，而返回的是U类型；
也就是说返回的类型与输入的第一个参数类型是一样的，而输入的第二个参数类型与Stream中元素类型是一样的。
-- combiner: （组合器）其类型是BinaryOperator，支持的是对U类型的对象进行操作；
擅长的是生成一个值

注意：非并行流下第三个参数没用。流中元素只有1个的情况下，即使指定parallel也不会走并行。
使用三个参数的reduce，务必注意线程安全。

```

字符串拼接、数组的sum、min、max、average都是特殊的reduce。

```java
	public void testReduce() {
        List<String> list = Arrays.asList("a11", "a2", "c1", "d1");

        // 1个参数 -> Optional<T> reduce(BinaryOperator<T> accumulator), 筛选长度最长的元素
        Optional<String> reduce = list.stream().reduce((s1, s2) -> s1.length() >= s2.length() ? s1 : s2);
        System.out.println(reduce.get());

        // 2个参数 -> T reduce(T identity, BinaryOperator<T> accumulator) s1表示的初始值 "@", 其返回类型取决于初始值类型
        String reduce1 = list.stream().reduce("@", (s1, s2) -> s1 + " = " + s2);
        System.out.println(reduce1);

        // 3个参数 -> <U> U reduce(U identity,BiFunction<U, ? super T, U> accumulator,BinaryOperator<U> combiner), 求和
        Integer reduce2 = list.stream().reduce(1,
                (sum, str) -> sum + str.length(),
                (s1, s2) -> {
                    return s1 + s2;
                });
        System.out.println(reduce2);

        // 字符串拼接
        String concat = Stream.of("A", "B", "C", "D").reduce("", String::concat);
        System.out.println("字符串拼接concat = " + concat);

        // 最小值
        double minValue = Stream.of(-1.5, 1.0, -3.0, -2.0).reduce(Double.MAX_VALUE, Double::min);
        System.out.println("最小值minVal = " + minValue);

        // 有起始值
        int sumValue = Stream.of(1, 2, 3, 4).reduce(0, Integer::sum);
        System.out.println("有起始值 sumValue = " + sumValue);

        // 无起始值
        int sumValue0 = Stream.of(1, 2, 3, 4).reduce(Integer::sum).get();
        System.out.println("无起始值 sumValue0 = " + sumValue0);
        // parallel()
        Integer reduce3 = list.stream().parallel().
                reduce(1,
                        (sum, str) -> sum + str.length(),
                        (s1, s2) -> {
                            System.out.println("thread name = " + Thread.currentThread().getName() + ", s1 = " + s1 + " ,s2 = " + s2);
                            return s1 + s2;
                        });
        System.out.println(reduce3);
    }
```

**Collectors**

Collectors.groupingBy(Function<? super T, ? extends K> classifier, Collector<? super T, A, D> downstream) 和 partitioningBy

```java
public void testGroupBy() {
    List<Employee> employees = new ArrayList<>();
    employees.add(new Employee("A", new Department("开发")));
    employees.add(new Employee("B", new Department("开发")));
    employees.add(new Employee("C", new Department("策划")));
    employees.add(new Employee("D", new Department("策划")));
    employees.add(new Employee("E", new Department("营销")));
    // groupingBy
    Map<Department, List<Employee>> byDept = employees.stream()
            .collect(Collectors.groupingBy(Employee::getDepartment));
    System.out.println(byDept);

    // 以部门名称分组
    Map<String, List<Employee>> collect = employees.stream()
            .collect(Collectors.groupingBy(e -> e.getDepartment().getName()));
    System.out.println(collect);


    // 按照部门对员工进行分组，并且只保留员工的名字
    Map<String, List<String>> collect1 = employees.stream()
            .collect(Collectors.groupingBy(e -> e.getDepartment().getName(), Collectors.mapping(Employee::getName, Collectors.toList())));
    System.out.println(collect1);
}
```



参考文章：https://objcoding.com/2019/03/04/lambda/#stream-pipelines