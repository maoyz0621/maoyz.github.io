# JAVA关键字-volatile

- volatile关键字的作用是什么?
- volatile能保证原子性吗? **不能保证原子性，只能保证volatile变量的单次的读/写操作具有原子性。**
- 之前32位机器上共享的long和double变量的为什么要用volatile? 现在64位机器上是否也要设置呢?**不需要**
- i++为什么不能保证原子性?
- volatile是如何实现可见性的? **内存屏障。**
- volatile是如何实现有序性的? **happens-before**等
- 说下volatile的应用场景?

## volatile

### 作用及实现原理

轻量级的synchronized

- 禁止指令重排序
- 保证内存可见性

- 不能保证原子性，只能保证volatile变量的单次的读/写操作具有原子性。

#### 指令重排

实例化一个对象其实可以分为三个步骤：

- 分配内存空间。
- 初始化对象。
- 将内存空间的地址赋值给对应的引用。

但是由于操作系统可以**对指令进行重排序**，所以上面的过程也可能会变成如下过程：

- 分配内存空间。
- 将内存空间的地址赋值给对应的引用。
- 初始化对象

如果是这个流程，多线程环境下就可能将一个未初始化的对象引用暴露出来，从而导致不可预料的结果。因此，为了防止这个过程的重排序，我们需要将变量设置为`volatile`类型的变量。

##### happens-before关系

对一个 volatile域的写，happens-before 于任意后续对这个 volatile 域的读。



##### 禁止重排序

为了性能优化，JMM 在不改变正确语义的前提下，会允许编译器和处理器对指令序列进行重排序。JMM 提供了**内存屏障**阻止这种重排序。

Java编译器会在生成指令系列时在适当的位置会插入**内存屏障**指令来禁止特定类型的处理器重排序。

- 在每个 volatile 写操作的前面插入一个 StoreStore 屏障。
- 在每个 volatile 写操作的后面插入一个 StoreLoad 屏障。
- 在每个 volatile 读操作的后面插入一个 LoadLoad 屏障。
- 在每个 volatile 读操作的后面插入一个 LoadStore 屏障。

> volatile  写是在前面和后面分别插入内存屏障
>
> volatile  读操作是在后面插入两个内存屏障。

| 内存屏障        | 说明                                                        |
| --------------- | ----------------------------------------------------------- |
| StoreStore 屏障 | 禁止上面的普通写和下面的 volatile 写重排序。                |
| StoreLoad 屏障  | 防止上面的 volatile 写与下面可能有的 volatile 读/写重排序。 |
| LoadLoad 屏障   | 禁止下面所有的普通读操作和上面的 volatile 读重排序。        |
| LoadStore 屏障  | 禁止下面所有的普通写操作和上面的 volatile 读重排序。        |

|            volatile  写            |            volatile  读            |
| :--------------------------------: | :--------------------------------: |
| ![](..\images\Lock\内存屏障-1.png) | ![](..\images\Lock\内存屏障-2.png) |



#### 保证可见性

一个线程修改了共享变量值，而另一个线程却看不到。

引起可见性问题的主要原因是每个线程拥有自己的一个高速缓存区——线程工作内存。

**内存可见性，基于内存屏障（Memory Barrier）实现**



#### 不能保证原子性

只能保证volatile变量的单次的读/写操作具有原子性。

> i++为什么不能保证原子性?

本质上i++是读、写两次操作。

- 读取i的值
- 对i加1
- 将i的值写回内存

volatile是无法保证这三个操作是具有原子性的，我们可以通过`AtomicInteger`或者`synchronized`来保证+1操作的原子性。

> 共享的long和double变量为什么要用volatile?

因为`long`和`double`两种数据类型的操作可分为高32位和低32位两部分，因此普通的`long`或`double`类型读/写可能不是原子的。因此，鼓励大家将共享的`long`和`double`变量设置为`volatile`类型，这样能保证任何情况下对`long`和`doubl`e的单次读/写操作都具有原子性。

PS：虚拟机都选择把 64 位数据的读写操作作为原子操作来对待，因此我们在编写代码时一般不把`long `和 `double` 变量专门声明为`volatile`多数情况下也是不会错的。



### 应用场景

- 状态标识

- 双重检查(double-checked)



通过**内存屏障**来防止指令被重排序?

> 在每个volatile写操作前插入StoreStore屏障，在写操作后插入StoreLoad屏障；
> 在每个volatile读操作前插入LoadLoad屏障，在读操作后插入LoadStore屏障；



用途：

​		结合CAS，保证原子性（AtomicXxx），常用于多线程环境下的单次操作。

用volatile修饰long和double可以保证其操作原子性。




## DCL

Double Check Lock双检锁

```java
public class SingletonLazy {

    /**
     * 因为这个变量要在静态方法中使用，所以需要加上static修饰
     * static对象,定义成null
     * volatile 可见性
     */
    private volatile static SingletonLazy instance = null;

    /**
     * 构造方法私有化
     */
    private SingletonLazy() {
    }

    /**
     * 提供获取实例的方法
     * 定义成static
     * 双重检查加锁
     */
    public static SingletonLazy getInstance() {
        //　判断存储实例的变量是否有值
        if (instance == null) {
            //　同步块，线程安全的创建实例
            synchronized (SingletonLazy.class) {
                //　再次检查实例是否存在，如果不存在才真的创建实例
                if (instance == null) {
                    //　如果没有，就创建一个类实例，并把值赋值给存储类实例的变量
                    instance = new SingletonLazy();
                }
            }
        }
        //　如果有值，那就直接使用
        return instance;
    }
}
```



## JSR内存屏障（Memory Barrier）

Java内存屏障主要有Load和Store两类。 
对Load Barrier来说，在读指令前插入读屏障，可以让高速缓存中的数据失效，重新从主内存加载数据 
对Store Barrier来说，在写指令之后插入写屏障，能让写入缓存的最新数据写回到主内存

| 屏障类型   | 指令示例                 | 说明                                                         |
| ---------- | ------------------------ | ------------------------------------------------------------ |
| LoadLoad   | Load1;Loadload;Load2     | 在Load2及后续读取操作要读取的数据被访问前，保证Load1要读取的数据被读取完毕 |
| StoreStore | Store1;StoreStore;Store2 | 在Store2及后续写入操作执行前，保证Store1的写入操作对其它处理器可见 |
| LoadStore  | Load1;LoadStore;Store2   | 在Store2及后续写入操作被刷出前，保证Load1要读取的数据被读取完毕 |
| StoreLoad  | Store1;StoreLoad;Load2   | 在Load2及后续所有读取操作执行前，保证Store1的写入对所有处理器可见 |



- LoadLoad 屏障

  序列：Load1 -> Loadload -> Load2 

- StoreStore 屏障

  序列：Store1 -> StoreStore -> Store2 

- LoadStore 屏障

  序列：Load1 -> LoadStore -> Store2 

- StoreLoad 屏障

  序列：Store1 -> StoreLoad ->  Load2
  
  **它的开销是四种屏障中最大的。在大多数处理器的实现中，这个屏障是个万能屏障，兼具其它三种内存屏障的功能**


