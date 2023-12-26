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

+ 将Array转换List,使用`Arrays.asList(T... a)`，此时获取的List是不可变List（`java.util.Arrays.ArrayList`），无法进行操作，否则抛出`UnsupportedOperationException`；

+ 原因：此时的ArrayList是Arrays中的静态内部类，同我们平常使用的ArrayList（`java.util.ArrayList`）不是同一个ArrlyList。

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

> ROUND_HALF_UP  向（距离）最近的一边舍入，除非两边（的距离）是相等，如果是这样，向上舍入, 1.55保留一位小数结果为1.6
> ROUND_HALF_DOWN   向（距离）最近的一边舍入，除非两边（的距离）是相等，如果是这样，向下舍入, 例如1.55 保留一位小数结果为1.5
> ROUND_HALF_EVEN  奇进偶不进，除非两边（的距离）是相等，如果是这样，如果保留位上的数字是奇数，使用ROUND_HALF_UP ，如果是偶数，使用ROUND_HALF_DOWN
> ROUND_UP    向远离0的方向舍入



```java
/**
 * 银行家算法
 * 舍去位数值 < 5 直接舍去
 * 舍去位数值 > 5 直接进位
 * 舍去位数值 = 5
 * 1、5后面还有其他非0数值，进位
 * 2、5后面 = 0，看前一位是偶数舍，奇数进位
 */
BigDecimal a = new BigDecimal("5.54");
System.out.println(a.setScale(1, RoundingMode.HALF_EVEN));  // 5.5

BigDecimal a2 = new BigDecimal("1.66");
System.out.println(a2.setScale(1, RoundingMode.HALF_EVEN));   // 1.7

BigDecimal a4 = new BigDecimal("1.06");

System.out.println(a4.setScale(1, RoundingMode.HALF_EVEN));  // 1.1


BigDecimal a1 = new BigDecimal("2.450");
System.out.println(a1.setScale(1, RoundingMode.HALF_EVEN));   // 2.4

BigDecimal a10 = new BigDecimal("2.550");
System.out.println(a10.setScale(1, RoundingMode.HALF_EVEN));  // 2.6

BigDecimal a11 = new BigDecimal("2.551");
System.out.println(a11.setScale(1, RoundingMode.HALF_EVEN));  // 2.6
BigDecimal a12 = new BigDecimal("2.555");
System.out.println(a12.setScale(1, RoundingMode.HALF_EVEN));  // 2.6


BigDecimal a3 = new BigDecimal("1.25");
System.out.println(a3.setScale(1, RoundingMode.HALF_EVEN));  // 1.2
BigDecimal a5 = new BigDecimal("1.55");
System.out.println(a5.setScale(1, RoundingMode.HALF_EVEN));  // 1.6
```

## 包装类型



包装类型参与计算时，谨慎null，原因：xxxValue，拆箱



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



## 枚举类

### 枚举类线程安全

枚举类编译为字节码后，**final**修饰的普通类，并且**extends java.lang.Enum<E extends Enum<E>>**，所有的属性被**static**和**final**修饰。在项目启动时，JVM加载并初始化。



Could not transfer artifact com.google.zxing:core:pom:3.3.2 from/to alimaven (https://maven.aliyun.com/repository/public): java.lang.RuntimeException: Unexpected error: java.security.InvalidAlgorithmParameterException: the trustAnchors parameter must be non-empty

InvalidAlgorithmParameterException：无效的算法参数异常



open-jdk会出现这样的问题，将jdk8中的cacerts文件拷贝到\security中

maven相关

1. 清除所有的lasted文件

```
#进入cmd命令行

cd /repo
for /r %i in (*.lastUpdated) do del %i
```

## 时区问题

```java
import java.text.DateFormat;
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.TimeZone;

public class TimeZoneTest {

    public static final String TIME_ZONE_GMT = "GMT+00:00";
    public static final String TIME_ZONE_BEIJING = "GMT+08:00";
    public static ThreadLocal<DateFormat> threadLocal = ThreadLocal.withInitial(() -> new SimpleDateFormat(DateFormatUtil.PATTERN_DEFAULT_ON_SECOND));

    /**
     * 获取日期对应时区时间
     *
     * @param pattern   格式化
     * @param timeZone  时区
     * @param timestamp 时间戳
     * @return String
     */
    public static String dateFormatFromTimestamp(String pattern, TimeZone timeZone, long timestamp) {
        DateFormat dateFormat = new SimpleDateFormat(pattern);
        dateFormat.setTimeZone(timeZone);
        return dateFormat.format(new Date(timestamp));
    }

    /**
     * 获取日期对应时区时间
     *
     * @param pattern   格式化
     * @param timeZone  时区
     * @param timestamp 时间戳
     * @return String
     */
    public static String dateFormatFromTimestamp(String pattern, String timeZone, long timestamp) {
        DateFormat dateFormat = new SimpleDateFormat(pattern);
        dateFormat.setTimeZone(TimeZone.getTimeZone(timeZone));
        return dateFormat.format(new Date(timestamp));
    }

    /**
     * Date表示当地的时间，即加上时区展示的时间
     *
     * @param timestamp 时间戳
     * @return Date
     */
    public static Date parseDate(long timestamp) {
        return new Date(timestamp);
    }

    public static Date parseDate(String date) throws ParseException {
        return parseDate(DateFormatUtil.PATTERN_DEFAULT_ON_SECOND, TimeZone.getTimeZone(TIME_ZONE_BEIJING), date);
    }

    /**
     * @param pattern  格式化
     * @param timeZone 时区
     * @param date     字符串时间
     * @return Date
     * @throws ParseException
     */
    public static Date parseDate(String pattern, TimeZone timeZone, String date) throws ParseException {
        DateFormat dateFormat = new SimpleDateFormat(pattern);
        dateFormat.setTimeZone(timeZone);
        return dateFormat.parse(date);
    }

    /**
     * @param pattern  格式化
     * @param timeZone 时区
     * @param date     字符串时间
     * @return Date
     * @throws ParseException
     */
    public static Date parseDate(String pattern, String timeZone, String date) throws ParseException {
        DateFormat dateFormat = new SimpleDateFormat(pattern);
        dateFormat.setTimeZone(TimeZone.getTimeZone(timeZone));
        return dateFormat.parse(date);
    }

    /**
     * 获取日期对应的0时区时间
     *
     * @param date Date
     * @return String
     */
    public static String dateFormat(Date date) {
        return dateFormat(DateFormatUtil.PATTERN_DEFAULT_ON_SECOND, TimeZone.getTimeZone(TIME_ZONE_GMT), date);
    }

    /**
     * 获取日期对应时区时间
     *
     * @param pattern  格式化
     * @param timeZone 时区
     * @param date     Date
     * @return String
     */
    public static String dateFormat(String pattern, String timeZone, Date date) {
        DateFormat dateFormat = new SimpleDateFormat(pattern);
        dateFormat.setTimeZone(TimeZone.getTimeZone(timeZone));
        return dateFormat.format(date);
    }

    /**
     * 获取日期对应时区时间
     *
     * @param pattern  格式化
     * @param timeZone 时区
     * @param date     Date
     * @return String
     */
    public static String dateFormat(String pattern, TimeZone timeZone, Date date) {
        DateFormat dateFormat = new SimpleDateFormat(pattern);
        dateFormat.setTimeZone(timeZone);
        return dateFormat.format(date);
    }

    /**
     * 获取当前时间戳
     *
     * @return long
     */
    public static long getCurrentTimeMillis() {
        return System.currentTimeMillis();
    }

    /**
     * 获取当前时间戳
     *
     * @return long
     */
    public static long getCurrentTime() {
        return getCurrentTimeMillis() / 1000;
    }

    public static void main(String[] args) throws ParseException {
        // Wed Nov 11 22:43:44 CST 2020  ，也就是东8区的时间
        System.out.println(parseDate(getCurrentTimeMillis()));
        // 0时区时间：2020-11-11 14:43:44
        System.out.println("0时区时间：" + dateFormatFromTimestamp(DateFormatUtil.PATTERN_DEFAULT_ON_SECOND, TIME_ZONE_GMT, getCurrentTimeMillis()));
        // 东8区时间：2020-11-11 22:43:44
        System.out.println("东8区时间：" + dateFormatFromTimestamp(DateFormatUtil.PATTERN_DEFAULT_ON_SECOND, TIME_ZONE_BEIJING, getCurrentTimeMillis()));
    }
}
```

- System.currentTimeMillis()获取当前的时间戳，是格林时间（GMT+00:00）的时间戳，不包含时区；
- 平常使用的new  Date()表示的时间是当地的时间，也就是加上时区（GMT+08:00）之后所呈现的时间

