# JAVA高阶知识

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
     *
     * @return SingletonLazy
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

- LoadLoad 屏障

  序列：Load1 -> Loadload -> Load2 

- StoreStore 屏障

  序列：Store1 -> StoreStore -> Store2 

- LoadStore 屏障

  序列：Load1 -> LoadStore -> Store2 

- StoreLoad 屏障

  序列：Store1 -> StoreLoad ->  Load2



## Happens - Before原则

导致多线程间可见性问题的两个“罪魁祸首”是*CPU缓存*和*重排序*，JMM定义了Happens-Before原则：**对于两个操作A和B，这两个操作可以在不同的线程中执行。如果A Happens-Before B，那么可以保证，当A操作执行完后，A操作的执行结果对B操作是可见的。**

1. 程序顺序规则
2. 锁定规则
3. volatile变量规则
4. 线程启动规则
5. 线程结束规则
6. 中断规则
7. 终结器规则
8. 传递性规则

|                                      |
| :----------------------------------: |
| ![](.\image\Java\Happens-Before.jpg) |

如图所示，这里的unlock M和lock M就是划分程序的分割线。在这里，红色区域和绿色区域的代码内部是可以进行重排序的，但是unlock和lock操作是不能与它们进行重排序的。即第一个图中的红色部分必须要在unlock M指令之前全部执行完，第二个图中的绿色部分必须全部在lock M指令之后执行。并且在第一个图中的unlock M指令处，红色部分的执行结果要全部刷新到主存中，在第二个图中的lock M指令处，绿色部分用到的变量都要从主存中重新读取。
在程序中加入分割线将其划分成多个程序块，虽然在程序块内部代码仍然可能被重排序，但是保证了程序代码在宏观上是有序的。并且可以确保在分割线处，CPU一定会和主内存进行交互。Happens-Before原则就是定义了程序中什么样的代码可以作为分隔线。并且无论是哪条Happens-Before原则，它们所产生分割线的作用都是相同的。



## volatile

- 禁止指令重排序
- 保证内存可见性

- 不能保证原子性

通过**内存屏障**来防止指令被重排序

在每个volatile写操作的前面插入一个StoreStore屏障。
在每个volatile写操作的后面插入一个StoreLoad屏障。
在每个volatile读操作的后面插入一个LoadLoad屏障。
在每个volatile读操作的后面插入一个LoadStore屏障。