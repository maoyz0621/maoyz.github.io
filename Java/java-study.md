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



## 包装类型



包装类型参与计算时，谨慎null，原因：xxxValue



## String



### 1. String创建实例

```java
String s = new String("xyz");
```

在运行时涉及几个String实例？

+ 答案：两个，一个是字符串字面量"xyz"所对应的、驻留（intern）在一个全局共享的字符串常量池中的实例，另一个是通过new String(String)创建并初始化的、内容与"xyz"相同的实例



```java
String s1 = new String("xyz");  
String s2 = new String("xyz"); 
```



参考文章：https://www.iteye.com/blog/rednaxelafx-774673



### 2. Java中，关于String类型的变量和常量做“+”运算时发生了什么？

```java
// 1
String s1 = "a" + "bc";
String s2 = "ab" + "c";
System.out.println(s1 == s2);   // true, 其实只创建了一个 "abc" 字符串对象，且位于字符串常量池中。

// 2
String a = "a";
String bc = "bc";

// String s2 = new StringBuilder().append(a).append(b).toString();
String s3 = a + bc;
System.out.println(s3 == s2);   // false

// 3
final String a0 = "a";
final String bc0 = "bc";

String s4 = a0 + bc0;
System.out.println(s4 == s2);   // true
```

1. 第一种情况：

   编译优化，优化手段：常量折叠，

   s1 = "a" + "bc" -> s1 = "abc" -> ldc "abc"

   s2 = "ab" + "c" -> s2 = "abc" -> ldc "abc"

   编译之后实则是：

   ```java
   String s1 = "abc";
   String s2 = "abc";
   ```

2. 第二种情况：

   优化手段，用StringBuilder来进行字符串拼接，

   s2 = a+bc -> s2 = new StringBuilder(a).append(bc).toString()

3. 第三种情况：

   final修饰的String，

   s4 = a0 + bc0 -> s4 = "abc" -> ldc "abc"

参考文章：https://www.zhihu.com/question/35014775

## 关键字

### 1. final

final域，编译器和处理器遵守两个重排序规则：

1. 在构造函数内对一个 final 域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。
2. 初次读一个包含 final 域的对象的引用，与随后初次读这个 final 域，这两个操作之间不能重排序。

### volatile

### sync

### static

