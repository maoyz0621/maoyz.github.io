# 语法

## for循环起别名

别名：deep
```
deep: for (int i = 0; i < 10; i++) {
    System.out.println(i);

    for (int j = 0; j < 10; j++) {
        if (j > 5){
            // 此时跳出外部循环
            break deep;
        }
    }
}
```

## Array

+ 将Array转换List,使用`Arrays.asList(T... a)`,此时获取的List是不可变List,无法进行操作,否则抛出`UnsupportedOperationException`；

+ 原因：此时的ArrayList是Arrays中的静态内部类，同我们平常使用的ArrayList不是同一个ArrlyList。

  ```
  public class Arrays {
      public static <T> List<T> asList(T... a) {
      	// 注意此时的Array
          return new ArrayList<>(a);
      }
      
      private static class ArrayList<E> extends AbstractList<E>
          implements RandomAccess, java.io.Serializable
      {
      	// 重写了set()
      	@Override
      	public E set(int index, E element) {
              E oldValue = a[index];
              a[index] = element;
              return oldValue;
          }
          
          // AbstractList提供
          public void add(int index, E element) {
          	throw new UnsupportedOperationException();
      	}
          
          // AbstractList提供
          public E remove(int index) {
              throw new UnsupportedOperationException();
          }
          
      }
  }
  ```


## Map

+ Map中putIfAbsent()和put()的区别:
    1) 使用put()方法添加键值对，如果map集合中没有该key对应的值，则直接添加，并返回null，如果已经存在对应的值，则会覆盖旧值，value为新的值。

    2) 使用putIfAbsent()方法添加键值对，如果map集合中没有该key对应的值，则直接添加，并返回null，如果已经存在对应的值，则依旧为原来的值。

