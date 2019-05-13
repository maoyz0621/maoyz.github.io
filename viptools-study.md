####  一 依赖来源
```
    <!-- 唯品会提供的工具类 start-->
    <dependency>
        <groupId>com.vip.vjtools</groupId>
        <artifactId>vjkit</artifactId>
        <version>${vjkit_version}</version>
    </dependency>
    <!-- optional for BeanMapper -->
    <dependency>
        <groupId>net.sf.dozer</groupId>
        <artifactId>dozer</artifactId>
        <version>5.5.1</version>
        <optional>true</optional>
    </dependency>
    <!-- 唯品会提供的工具类 end-->
```

#### 二 collection

##### 1 ArrayUtil
- 1. 创建Array的函数
- 2. 数组的乱序与contact相加
- 3. 从Array转换到Guava的底层为原子类型的List
- 4. JDK Arrays的其他函数，如sort(), toString() 请直接调用
- 5. Common Lang ArrayUtils的其他函数，如subarray(),reverse(),indexOf(), 请直接调用

>  + 将传入的数组乱序
    `huffle(T[] array)`
    `shuffle(T[] array, Random random)`

>  + 将Array转List
    `asList(T... a)`
    `intAsList(int... backingArray)`
    `longAsList(long... backingArray)`
    `doubleAsList(double... backingArray)`

> + 添加元素到数组
    `concat(@Nullable T element, T[] array)`添加元素到数组头;
    `concat(T[] array, @Nullable T element)`添加元素到数组末尾

##### 2 ListUtil
- 1. 常用函数(如是否为空，sort/binarySearch/shuffle/reverse(via JDK Collection)
- 2. 各种构造函数(from guava and JDK Collection)
- 3. 各种扩展List类型的创建函数
- 4. 集合运算：交集，并集, 差集, 补集，from Commons Collection，但对其不合理的地方做了修正

> + 初始化List
    `newArrayList(T... elements)`
    `newArrayList(Iterable<T> elements)`
    `newArrayListWithCapacity(int initSize)`穿件初始容量
    `emptyList()`返回一个空的结构特殊的List，节约空间
    `singletonList(T o)`返回只含一个元素但结构特殊的List，节约空间
    `unmodifiableList(List<? extends T> list)`
    `synchronizedList(List<T> list)`返回包装后同步的List，所有方法都会被synchronized原语同步,用于CopyOnWriteArrayList与 ArrayDequeue均不符合的场景

> +  List排序和查找
    `sort(List<T> list)`
    `sort(List<T> list, Comparator<? super T> c)`
    `shuffle(List<?> list)`洗牌
    `binarySearch(List<? extends Comparable<? super T>> sortedList, T key)`
    `reverse(final List<T> list)`
    `sortReverse(List<T> list)`

> + List计算
    `union`并集
    `intersection`交集
    `difference`差集
    `disjoint`补集

##### 3 MapUtil



##### 3 IdUtil
> 1. 计算UUID
    `fastUUID`

##### BeanMapper
> 1. 使用Dozer提供的Mapper
    `map(S source, Class<D> destinationClass)`简单的复制出新类型对象
    `mapList(Iterable<S> sourceList, Class<D> destinationClass)`简单的复制出新对象ArrayList
    `mapArray(final S[] sourceArray, final Class<D> destinationClass)`简单复制出新对象数组

##### JsonMapper
> 1. JsonMapper的创建
    `JsonMapper.INSTANCE`
    `JsonMapper.defaultMapper()`
    `JsonMapper.nonNullMapper()`非null属性
    `JosnMapper.nonEmptyMapper()`非Null且非Empty

> 2. toJson
    `toJson(Object object)`Object可以是POJO，也可以是Collection或数组。 如果对象为Null, 返回"null". 如果集合为空集合, 返回"[]"

> 3. fromJson
    `fromJson(@Nullable String jsonString, Class<T> clazz)`如果JSON字符串为Null或"null"字符串, 返回Null. 如果JSON字符串为"[]", 返回空集合
    `fromJson(@Nullable String jsonString, JavaType javaType)`反序列化复杂Collection如List<Bean>

> 4. update
    `update(String jsonString, Object object)`当JSON里只含有Bean的部分属性時，更新一個已存在Bean，只覆盖該部分的属性

##### XmlMapper
> 1. bean转xml
    `toXml(Object root, Class clazz, String encoding)`

```
      User user = new User();
      user.setId(1L);
      user.setName("calvin");
      user.setPassword("123456");

      user.getInterests().add("movie");
      user.getInterests().add("sports");

      String xml = XmlMapper.toXml(user, "UTF-8");
      // <?xml version="1.0" encoding="UTF-8" standalone="yes"?>
      // <user id="1">
      //     <name>calvin</name>
      //     <interests>
      //         <interest>movie</interest>
      //         <interest>sports</interest>
      //     </interests>
      // </user>
      System.out.println("Jaxb Object to Xml result:\n" + xml);

      ///////////////////////////////////////////////////////////

      User user1 = new User();
      user1.setId(1L);
      user1.setName("calvin");
      user1.getInterests().add("movie1");
      user1.getInterests().add("sports1");

      User user2 = new User();
      user2.setId(2L);
      user2.setName("kate");
      user2.getInterests().add("movie2");
      user2.getInterests().add("sports2");

      List<User> userList = ListUtil.newArrayList(user1, user2);

      String xml = XmlMapper.toXml(userList, "userList", User.class, "UTF-8");
      // <?xml version="1.0" encoding="UTF-8" standalone="yes"?>
      // <userList>
      //     <user id="1">
      //         <name>calvin</name>
      //         <interests>
      //             <interest>movie1</interest>
      //             <interest>sports1</interest>
      //         </interests>
      //     </user>
      //     <user id="2">
      //         <name>kate</name>
      //         <interests>
      //             <interest>movie2</interest>
      //             <interest>sports2</interest>
      //         </interests>
      //     </user>
      // </userList>
      System.out.println("Jaxb Object List to Xml result:\n" + xml);
```

##### Number

##### MathUtil
> 1. 2的倍数
    `nextPowerOfTwo()`
    `previousPowerOfTwo`

> 2. 取模与开方
    `mod()`
    `pow()`
    `sqrt()`

##### MoneyUtil
> 1. 分 --> 元, 元 --> 分
    `fen2yuan()`
    `yuan2fen()`

> 2. 金额转字符串
    `format(double number, String pattern)`1234.00
    `prettyFormat(double number)`33,999,999,932.33

> 3. 字符串转金额
    `parsePrettyString("1,234.00")`
    `parseString("1234.00")`

##### RandomUtil
> 1. Random实例
    `secureRandom()`
    `threadLocalRandom()`

> 2. 生成随机数
    `nextInt(int min, int max)`返回min到max的随机Int, 使用ThreadLocalRandom
    `nextInt(Random random, int max)`返回0到max的随机Int, 可传入SecureRandom或ThreadLocalRandom
    `randomStringFixLength(int length)`随机字母或数字，固定长度
    `randomStringRandomLength(int minLength, int maxLength)`随机字母或数字，随机长度
    `randomLetterFixLength(int length)`随机字母，固定长度
    `randomLetterRandomLength(int minLength, int maxLength)`随机字母，随机长度

##### Time
###### DateFormatUtil
> 1. 采用Apache Common Lang中线程安全, 性能更佳的FastDateFormat
    `formatDate(@NotNull String pattern, @NotNull Date date)`
    `parseDate(@NotNull String pattern, @NotNull String dateString)`
    `formatFriendlyTimeSpanByNow(@NotNull Date date)`打印用户友好的，与当前时间相比的时间差，如刚刚，5分钟前，今天XXX，昨天XXX
    `formatFriendlyTimeSpanByNow(long timeStampMillis)`打印用户友好的，与当前时间相比的时间差，如刚刚，5分钟前，今天XXX，昨天XXX
    `formatDuration()`

###### DateUtil
> 1. 时间比较
    `isSameDay()`同一天
    `isSameTime()`同意时刻
    `isBetween()`时间范围内

> 2. 时间操作
    `addMonths()`
    `subMonths()`
    `addWeeks()`
    `subWeeks()`
    `addDays()`
    `subDays()`
    `addHours()`
    `subHours()`
    `addMinutes()`
    `subMinutes()`
    `addSeconds()`
    `subSeconds()`

> 3. 设置时间
    `setYears()`
    `setMonths()`
    `setDays()`
    `setHours()`
    `setMinutes()`
    `setSeconds()`
    `setMilliseconds()`

> 4. 获取日期的位置
    `getDayOfWeek()`获得日期是一周的第几天. 已改为中国习惯，1 是Monday，而不是Sundays
    `getDayOfYear()`
    `getWeekOfMonth`
    `getWeekOfYear()`

> 5. 获得往前往后的日期
    `beginOfYear()`
    `endOfYear()`
    `nextYear()`
    `beginOfMonth()`当前月份的开始时间
    `endOfMonth()`当前月份的结束时间
    `beginOfDate()`
    `endOfDate()`
    `nextDate()`
    `beginOfHour()`
    `endOfHour()`
    `nextHour()`
    `isLeapYear()`是否闰年

#### Concurrent
##### limiter
###### RateLimiterUtil
> +
    `create(double permitsPerSecond, double maxBurstSeconds, boolean filledWithToken)`
###### 采样器Sampler

###### TimeIntervalLimiter
