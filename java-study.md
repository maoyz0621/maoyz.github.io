# 语法

## for循环起别名

别名：deep
```java
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

  ```java
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

## 浮点计算

### double

```java
double a = 0.03D;
double b = 0.02D;
//0.05
System.out.println(a + b);
// 0.009999999999999998
System.out.println(a - b);
//6.0E-4
System.out.println(a * b);
System.out.println(a / b);
```



### BigDecimal

浮点类型确保精度，使用BigDecimal，传入字符串类型

```java
 // BigDecimal()  传入String类型才能精确计算
 BigDecimal a1 = new BigDecimal("0.03");
 BigDecimal b1 = new BigDecimal("0.02");
 // 0.05
 System.out.println(b1.add(a1));
 // 0.01
 System.out.println(a1.subtract(b1));
 // 0.0006
 System.out.println(a1.multiply(b1));
 System.out.println(a1.divide(b1, RoundingMode.HALF_UP));
```

#### 转化

double转BigDecimal时，不要直接传入double类型

```java
BigDecimal a2 = new BigDecimal(0.03d);
// 0.0299999999999999988897769753748434595763683319091796875
System.out.println("a2 = " + a2);   
BigDecimal b2 = new BigDecimal(0.02d);
// 0.0200000000000000004163336342344337026588618755340576171875
System.out.println("b2 = " + b2);   
```

那么如何转化？

+ 方法一：

```java
BigDecimal.valueOf(double d)
```

+ 方法二：

```java
BigDecimal b2 = new BigDecimal(0.02d + "");
System.out.println("b2 = " + b2);   // 0.02
```

BigDecimal转double

```java
doubleValue()
```



#### 值比较

在比较过程中使用compareTo()，返回值 -1 0 1，分别表示（前者和后者比较）小于 等于 大于

```java
BigDecimal a = new BigDecimal("1");
BigDecimal b = new BigDecimal("1.00");
BigDecimal c = new BigDecimal("1.01");
System.out.println(a.compareTo(b));    // 0
System.out.println(b.compareTo(c));    // -1
System.out.println(c.compareTo(a));    // 1
```

当我们使用equals()比较0和0.00的时候

```java
System.out.println(new BigDecimal("0").compareTo(new BigDecimal("0.00")));    // 0
System.out.println(new BigDecimal("0").equals(new BigDecimal("0.00")));    // false
```

#### 保留小数点和四舍五入

```java
setScale(int newScale, RoundingMode roundingMode)
```

