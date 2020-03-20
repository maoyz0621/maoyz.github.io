# CompletableFuture和ForkJoinPool 

CompletableFuture对Future任务的一个花式处理。

## CompletableFuture

| 功能                                                         | 提供方法                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| Future合并，生成新的Future                                   | allOf<br/>anyOf<br/>runAsync<br/>supplyAsync                 |
| Future添加后置处理动作                                       | thenAccept<br/>thenApply<br/>thenRun                         |
| 两个Future 任一或全部完成时，执行后置动作                    | applyToEither<br/>acceptEither<br/>thenAcceptBothAsync<br/>runAfterBoth<br/>runAfterEither |
| 当 Future 完成条件满足时，异步或同步执行后置处理动作         | thenApplyAsync<br/>thenRunAsync                              |
| Future处理顺序 thenCompose 协同存在依赖关系的 Future。<br>合并多个 Future的处理结果返回新的处理结果 | thenCombine                                                  |
| 异常处理 exceptionally ，如果任务处理过程中抛出了异常        |                                                              |



### 静态工厂方法



| 方法名                                               | 描述                                                         |
| ---------------------------------------------------- | ------------------------------------------------------------ |
| runAsync(Runnable runnable)                          | 使用ForkJoinPool.commonPool()作为它的线程池执行异步代码。    |
| runAsync(Runnable runnable, Executor executor)       | 使用指定的thread pool执行异步代码。                          |
| supplyAsync(Supplier<U> supplier)                    | 使用ForkJoinPool.commonPool()作为它的线程池执行异步代码，异步操作有返回值 |
| supplyAsync(Supplier<U> supplier, Executor executor) | 使用指定的thread pool执行异步代码，异步操作有返回值          |



### Complete

|                                     |                                  |
| ----------------------------------- | -------------------------------- |
| complete(T t)                       | 完成异步执行，并返回future的结果 |
| completeExceptionally(Throwable ex) | 异步执行不正常的结束             |



### 转换

#### map

|                                                              |                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| thenApply(Function<? super T,? extends U> fn)                | 接受一个Function<? super T,? extends U>参数用来转换CompletableFuture |
| thenApplyAsync(Function<? super T,? extends U> fn)           | 接受一个Function<? super T,? extends U>参数用来转换CompletableFuture，使用ForkJoinPool |
| thenApplyAsync(Function<? super T,? extends U> fn, Executor executor) | 接受一个Function<? super T,? extends U>参数用来转换CompletableFuture，使用指定的线程池 |



#### flatmap

|                                                              |                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| thenCompose(Function<? super T, ? extends CompletionStage<U>> fn) | 在异步操作完成的时候对异步操作的结果进行一些操作，并且仍然返回CompletableFuture类型。 |
| thenComposeAsync(Function<? super T, ? extends CompletionStage<U>> fn) | 在异步操作完成的时候对异步操作的结果进行一些操作，并且仍然返回CompletableFuture类型。使用ForkJoinPool。 |
| thenComposeAsync(Function<? super T, ? extends CompletionStage<U>> fn,Executor executor) | 在异步操作完成的时候对异步操作的结果进行一些操作，并且仍然返回CompletableFuture类型。使用指定的线程池。 |

多个CompletableFuture之间存在先后顺序。

```java
public void test() throws Exception {
    CompletableFuture<Double> future = CompletableFuture.supplyAsync(() -> "101")
            .thenComposeAsync(s -> {
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("thenCompose 1");
                return CompletableFuture.supplyAsync(() -> s + "100");
            }, executor)
            .thenCompose(s -> {
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("thenCompose 2");
                return CompletableFuture.supplyAsync(() -> Double.parseDouble(s));
            });

    // 101100.0
    System.out.println(future.get());
}
```

### 组合 thenCombine

|                                                              |                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| thenCombine(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn) | 当两个CompletableFuture都正常完成后，执行提供的fn，用它来组合另外一个CompletableFuture的结果 |
| thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn) | 当两个CompletableFuture都正常完成后，执行提供的fn，用它来组合另外一个CompletableFuture的结果。使用ForkJoinPool。 |
| thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn, Executor executor) | 当两个CompletableFuture都正常完成后，执行提供的fn，用它来组合另外一个CompletableFuture的结果。使用指定的线程池。 |

多个CompletableFuture之间是并行执行的，最后再将结果汇总。

```java
public void test() throws Exception {
    CompletableFuture<Double> future = CompletableFuture.supplyAsync(() -> {
        try {
            Thread.sleep(3000);
            System.out.println("thenCombine 1");
        } catch (InterruptedException e) {
        }
        return "101";
    }).thenCombine(
            CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(2000);
                    System.out.println("thenCombine 2");
                } catch (InterruptedException e) {
                }
                return "100";
            }), (s, i) -> {
                System.out.println("s = " + s);
                System.out.println("i = " + i);
                return Double.parseDouble(s + i);
            });

    // 101100.0
    System.out.println(future.get());
}
```
返回Void

|                                                              |                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| thenAcceptBoth(CompletionStage<? extends U> other, BiConsumer<? super T,? super U> action) | 当两个CompletableFuture都正常完成后，执行提供的action，用它来组合另外一个CompletableFuture的结果。 |
| thenAcceptBothAsync(CompletionStage<? extends U> other, BiConsumer<? super T,? super U> action) | 当两个CompletableFuture都正常完成后，执行提供的action，用它来组合另外一个CompletableFuture的结果。使用ForkJoinPool。 |
| thenAcceptBothAsync(CompletionStage<? extends U> other, BiConsumer<? super T,? super U> action, Executor executor) | 当两个CompletableFuture都正常完成后，执行提供的action，用它来组合另外一个CompletableFuture的结果。使用指定的线程池。 |

```java
public void test() throws Exception {
    CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> {
        try {
            Thread.sleep(3000);
            System.out.println("thenCombine 1");
        } catch (InterruptedException e) {
        }
        return "101";
    }).thenAcceptBoth(
            CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(2000);
                    System.out.println("thenCombine 2");
                } catch (InterruptedException e) {
                }
                return "100";
            }), (s, i) -> {
                System.out.println("s = " + s);
                System.out.println("i = " + i);
                System.out.println(Double.parseDouble(s + i));
            });

    // null
    System.out.println(future.get());
}
```



### 计算结果处理

#### 执行特定的Action

|                                                              |                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| whenComplete(BiConsumer<? super T,? super Throwable> action) | 当CompletableFuture完成计算结果时对结果进行处理，或者当CompletableFuture产生异常的时候对异常进行处理 |
| whenCompleteAsync(BiConsumer<? super T,? super Throwable> action) | 当CompletableFuture完成计算结果时对结果进行处理，或者当CompletableFuture产生异常的时候对异常进行处理。使用ForkJoinPool。 |
| whenCompleteAsync(BiConsumer<? super T,? super Throwable> action, Executor executor) | 当CompletableFuture完成计算结果时对结果进行处理，或者当CompletableFuture产生异常的时候对异常进行处理。使用指定的线程池。 |

```java
public void testWhenComplete() {

    String error = CompletableFuture.supplyAsync(() -> {
        try {
            Thread.sleep(2 * 1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        if (1 == 1) {
            throw new RuntimeException("Error");
        }
        return Thread.currentThread().getName() + " " + LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm"));
    }).whenComplete((s, e) -> {
        System.out.println("whenComplete result: " + s);
        System.out.println("whenComplete error: " + e);
    }).join();

    System.out.println(error);
}
```



#### 执行完Action后可以转换

|                                                              |                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| handle(BiFunction<? super T, Throwable, ? extends U> fn)     | 当CompletableFuture完成计算结果或者抛出异常的时候，执行提供的fn |
| handleAsync(BiFunction<? super T, Throwable, ? extends U> fn) | 当CompletableFuture完成计算结果或者抛出异常的时候，执行提供的fn，使用ForkJoinPool。 |
| handleAsync(BiFunction<? super T, Throwable, ? extends U> fn, Executor executor) | 当CompletableFuture完成计算结果或者抛出异常的时候，执行提供的fn，使用指定的线程池。 |

```java
public void testHandle() {
    String error = CompletableFuture.supplyAsync(() -> {
        try {
            Thread.sleep(2 * 1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        // if (1 == 1) {
        //     throw new RuntimeException("Error");
        // }
        return Thread.currentThread().getName() + " " + LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm"));
    }).handle((s, e) -> {
        if (e != null) {
            return "error";
        }
        return s;
    }).join();
    System.out.println(error);
}
```

#### 纯消费

|                                                              |                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| thenAccept(Consumer<? super T> action)                       | 当CompletableFuture完成计算结果，只对结果执行Action，而不返回新的计算值 |
| thenAcceptAsync(Consumer<? super T> action)                  | 当CompletableFuture完成计算结果，只对结果执行Action，而不返回新的计算值，使用ForkJoinPool。 |
| thenAcceptAsync(Consumer<? super T> action, Executor executor) | 当CompletableFuture完成计算结果，只对结果执行Action，而不返回新的计算值 |

```java
public void testAccept() {
    for (int i = 0; i < 1000; i++) {
        CompletableFuture.supplyAsync(() -> LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")))
                .thenAccept(s -> System.out.println(Thread.currentThread().getName() + " => now is " + s))
                .join();
    }
}
```



### Either

|                                                              |                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| acceptEither(CompletionStage<? extends T> other, Consumer<? super T> action) | 当任意一个CompletableFuture完成的时候，action这个消费者就会被执行。 |
| acceptEitherAsync(CompletionStage<? extends T> other, Consumer<? super T> action) |                                                              |
| acceptEitherAsync(CompletionStage<? extends T> other, Consumer<? super T> action, Executor executor) |                                                              |
| applyToEither(CompletionStage<? extends T> other, Function<? super T,U> fn) | 当任意一个CompletableFuture完成的时候，fn会被执行，它的返回值会当作新的CompletableFuture<U>的计算结果。 |
| applyToEitherAsync(CompletionStage<? extends T> other, Function<? super T,U> fn) |                                                              |
| applyToEitherAsync(CompletionStage<? extends T> other, Function<? super T,U> fn, Executor executor) |                                                              |



### Other

| 静态方法                           | 可以使用多个Future                                 |
| ---------------------------------- | -------------------------------------------------- |
| allOf(CompletableFuture<?>... cfs) | 在所有Future对象完成后结束，并返回一个future       |
| anyOf(CompletableFuture<?>... cfs) | 在任何一个Future对象结束后结束，并返回一个future。 |
|                                    |                                                    |

```java
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> "tony");
CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> "cafei");
CompletableFuture<String> future3 = CompletableFuture.supplyAsync(() -> "aaron");

CompletableFuture.allOf(future1, future2, future3)
    .thenApply(v ->Stream.of(future1, future2, future3)
               .map(CompletableFuture::join)
               .collect(Collectors.joining(" ")))
    .thenAccept(System.out::print);
```



### 异常处理

|                            |                                                              |
| -------------------------- | ------------------------------------------------------------ |
| exceptionally(Function fn) | 只有当CompletableFuture抛出异常的时候，才会触发这个exceptionally的计算，调用function计算值。 |



参考文章：https://juejin.im/post/59eae7636fb9a045117044c6



## ForkJoinPool 

