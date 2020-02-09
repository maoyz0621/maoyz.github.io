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
          0: new           #2                  // class java/lang/Thread
          3: dup
          4: new           #3                  // class com/myz/java/study/java8/lambda/MainAnonymousClassTest$1 /* 创建内部类 */
          7: dup
          8: invokespecial #4                  // Method com/myz/java/study/java8/lambda/MainAnonymousClassTest$1."<init>":()V
         11: invokespecial #5                  // Method java/lang/Thread."<init>":(Ljava/lang/Runnable;)V
         14: invokevirtual #6                  // Method java/lang/Thread.start:()V
         17: return
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
   
     private static void lambda$main$0();	/*Lambda表达式被封装成主类的私有方法，之后通过invokedynamic指令调用*/
       Code:
          0: getstatic     #6                  // Field java/lang/System.out:Ljava/io/PrintStream;
          3: ldc           #7                  // String Lambda实现Runnable
          5: invokevirtual #8                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
          8: return
   }
   ```

   

1. this指向

   对于匿名类，关键词 this 解读为匿名类本身，而对于 Lambda 表达式，关键词 this 为 Lambda 的外部类。

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

返回容器的可拆分迭代器

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

按照给定的`key`查询`Map`中对应的`value`，如果没有找到则返回`defaultValue`

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
    if (!Objects.equals(curValue, oldValue) ||
        (curValue == null && !containsKey(key))) {
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

