## Array

+ 将Array转换List,使用`Arrays.asList(T... a)`,此时获取的List是不可变List,无法进行操作,否则抛出`UnsupportedOperationException`


## Map

+ Map中putIfAbsent()和put()的区别:
    1) 使用put()方法添加键值对，如果map集合中没有该key对应的值，则直接添加，并返回null，
如果已经存在对应的值，则会覆盖旧值，value为新的值。

    2) 使用putIfAbsent()方法添加键值对，如果map集合中没有该key对应的值，则直接添加，并返
回null，如果已经存在对应的值，则依旧为原来的值。
