# JAVA并发包-锁-LockSupport

> 原文：https://pdai.tech/md/java/thread/java-thread-x-lock-LockSupport.html



- 为什么LockSupport也是核心基础类? AQS框架借助于两个类：**Unsafe**(提供CAS操作)和**LockSupport**(提供park/unpark操作)
- wait/notify和LockSupport的park/unpark实现同步?
- LockSupport.park()会释放锁资源吗? 那么Condition.await()呢?
- **Thread.sleep()、Object.wait()、Condition.await()、LockSupport.park()的区别? 重点**
- 如果在wait()之前执行了notify()会怎样?
- 如果在park()之前执行了unpark()会怎样?可以正常唤醒线程



## LockSupport

创建锁和其他同步类的基本线程阻塞。简而言之，当调用LockSupport.park时，表示当前线程将会等待，直至获得许可，当调用LockSupport.unpark时，必须把等待获得许可的线程作为参数进行传递，好让此线程继续运行

```java
private LockSupport() {} // Cannot be instantiated.
```

无法实例化

## park

```java
public static void park() {
    // 获取许可，设置时间为无限长，直到可以获取许可
    U.park(false, 0L);
}

public static void park(Object blocker) {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 设置Blocker
    setBlocker(t, blocker);
    // 获取许可
    U.park(false, 0L);
    // 重新可运行后再此设置Blocker
    setBlocker(t, null);
}
```

基于`sun.misc.Unsafe.park`

```java
public native void park(boolean isAbsolute, long time)
```

`LockSupport.park()`不会释放资源，只阻塞当前线程，线程处于休眠状态，在下列情况之前都会被阻塞：

1. 调用unpark，释放该线程的许可
2. 线程被中断
3. 设置的时间到了，并且，time为绝对时间，isAbsolute=true，否则，isAbsolute为false。当time为0时，表示无限等待，直到unpark发生

释放锁资源实际上是在`Condition`的`await()`方法中实现。

为什么要在此park函数中要调用两次setBlocker函数呢？ 原因其实很简单，调用park函数时，当前线程首先设置好parkBlocker字段，然后再调用Unsafe的park函数，此后，当前线程就已经阻塞了，等待该线程的unpark函数被调用，所以后面的一个setBlocker函数无法运行，unpark函数被调用，该线程获得许可后，就可以继续运行了，也就运行第二个setBlocker，把该线程的parkBlocker字段设置为null，这样就完成了整个park函数的逻辑。如果没有第二个setBlocker，那么之后没有调用park(Object blocker)，而直接调用getBlocker函数，得到的还是前一个park(Object blocker)设置的blocker，显然是不符合逻辑的。总之，必须要保证在park(Object blocker)整个函数执行完后，该线程的parkBlocker字段又恢复为null。所以，park(Object)型函数里必须要调用setBlocker函数两次。



## unpark

```java
public static void unpark(Thread thread) {
    if (thread != null) // 线程为不空
        U.unpark(thread); // 释放该线程许可
}
```

基于`sun.misc.Unsafe.unpark`

```java
public native void unpark(Object thread);
```

释放线程的许可，激活调用park后阻塞的线程（注意要确保这个线程依旧存活）

在park之前执行unpark，线程不会被阻塞，直接跳过park，据需执行后续内容。

> 唤醒线程，interrupt()起到的作用与unpark一样



## park/unpark实现线程同步

```java
public class LockSupportTest {

    /**
     * 使用park和unpark实现线程同步
     */
    public static void main(String[] args) {
        Thread thread = new Thread(new MyThread(Thread.currentThread()));
        thread.start();

        // 这样就限制性unpark，后执行park
        // try {
        //     Thread.sleep(3000L);
        // } catch (InterruptedException e) {
        //     e.printStackTrace();
        // }

        log.info("before park");
        LockSupport.park("LockSupportTest-blocker");
        log.info("after park");
    }

    static class MyThread implements Runnable {
        private Object obj;

        public MyThread(Object obj) {
            this.obj = obj;
        }

        @Override
        public void run() {
            log.info("start unpark");

            try {
                Thread.sleep(1000L);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            log.info("Blocker info: {}", LockSupport.getBlocker((Thread) obj));

            // 1. 唤醒线程：使用LockSupport.unpark
            LockSupport.unpark((Thread) obj);


            // 2. 唤醒线程：使用interrupt
            // ((Thread) obj).interrupt();

            try {
                Thread.sleep(500L);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            log.info("after unpark Blocker info: {}", LockSupport.getBlocker((Thread) obj));
            log.info("after unpark");
        }
    }
}
```

唤醒线程：使用LockSupport.unpark()、interrupt()

## 区别

### Thread.sleep()和Object.wait()的区别

- Thread.sleep()不会释放占有的锁，Object.wait()会释放占有的锁；
- Thread.sleep()必须传入时间，Object.wait()可传可不传，不传表示一直阻塞下去；
- Thread.sleep()到时间了会自动唤醒，然后继续执行；
- Object.wait()不带时间的，需要另一个线程使用Object.notify()唤醒；
- Object.wait()带时间的，假如没有被notify()，到时间了会自动唤醒，这时又分两种情况：一是立即获取到了锁，线程自然会继续执行；二是没有立即获取锁，线程进入同步队列等待获取锁；

> 最大的区别就是Thread.sleep()不会释放锁资源，Object.wait()会释放锁资源。



### Object.wait()和Condition.await()的区别

Object.wait()和Condition.await()的原理是基本一致的，不同的是Condition.await()底层是调用LockSupport.park()来实现阻塞当前线程的。

实际上，它在阻塞当前线程之前还干了两件事，一是把当前线程添加到条件队列中，二是“完全”释放锁，也就是让state状态变量变为0，然后才是调用LockSupport.park()阻塞当前线程。



### Thread.sleep()和LockSupport.park()的区别

- 两者都是阻塞当前线程的执行，且都不会释放当前线程占有的锁资源；
- Thread.sleep()没法从外部唤醒，只能自己醒过来；
- LockSupport.park()方法可以被另一个线程调用LockSupport.unpark()方法唤醒；
- Thread.sleep()方法声明上抛出了InterruptedException中断异常，所以调用者需要捕获这个异常或者再抛出；
- LockSupport.park()方法不需要捕获中断异常；
- Thread.sleep()本身就是一个native方法；
- LockSupport.park()底层是调用的Unsafe的native方法；



### Object.wait()和LockSupport.park()的区别

二者都会阻塞当前线程的运行

- Object.wait()方法需要在synchronized块中执行；
- LockSupport.park()可以在任意地方执行；
- Object.wait()方法声明抛出了中断异常，调用者需要捕获或者再抛出；
- LockSupport.park()不需要捕获中断异常；
- Object.wait()不带超时的，需要另一个线程执行notify()来唤醒，但不一定继续执行后续内容；
- LockSupport.park()不带超时的，需要另一个线程执行unpark()来唤醒，一定会继续执行后续内容；



## 总结

|          | Thread.sleep |         Object.wait          |      Condition.await       |  LockSupport.park  |
| :------: | :----------: | :--------------------------: | :------------------------: | :----------------: |
| 释放资源 |      N       |              Y               |             Y              |         N          |
|   唤醒   | 到达睡眠时间 |   Object.notify/notifyAll    | Condition.signal/signalAll | LockSupport.unpark |
| 使用条件 |      无      | 依赖synchronized，必须先wait |   依赖Lock，必须先await    | 任何，无需先后顺序 |

