# java并发包

## JUC (java.util.concurrent)

### 1. collections

### 1.1 List

#### CopyOnWriteArrayList

- 原理: 需要修改（增/删/改）列表中的元素时，不直接进行修改，而是先将列表Copy，然后在新的副本上进行修改，修改完成之后，再将引用从原列表指向新列表  

- 重要参数:  

```
    // 排它锁，用于数据修改
    final transient ReentrantLock lock = new ReentrantLock();
 
    // 内部数据组
    private transient volatile Object[] array;
```

- 主要方法  

1）add() 

首先会进行加锁，保证只有一个线程能进行修改；然后会创建一个新数组（大小为n+1），并将原数组的值复制到新数组，新元素插入到新数组的最后；最后，将字段array指向新数组  

```
    // this.lock 同一个对象引用同一把锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // 获取旧数据
        Object[] elements = getArray();
        int len = elements.length;
        // 数组拷贝，并且长度+1
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        // 将元素插入到新数组末尾
        newElements[len] = e;
        // 内部array引用指向新数组
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
```

2）get()  

```
    // 没有加锁，直接返回了内部数组对应索引位置的值：array[index]
    return (E) array[index];
```

3）remove()  

```
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        // 获取旧数组中的元素, 用于返回
        E oldValue = get(elements, index);
        // 需要移动多少个元素
        int numMoved = len - index - 1;
        // index位置刚好是最后一个元素
        if (numMoved == 0)                  
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            Object[] newElements = new Object[len - 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index, numMoved);
            setArray(newElements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
```

- 迭代

`COWIterator`的迭代是在`旧数组`上进行的，当创建迭代器的那一刻就确定了，所以迭代过程中不会抛出并发修改异常——ConcurrentModificationException。
另外，迭代器对象也不支持修改方法，全部会抛出UnsupportedOperationException异常。  

```
    public void remove() {
        throw new UnsupportedOperationException();
    }

    public void set(E e) {
        throw new UnsupportedOperationException();
    }

    public void add(E e) {
        throw new UnsupportedOperationException();
    }
```

- 缺点

1. 内存的使用  
由于CopyOnWriteArrayList使用了“写时复制”，所以在进行写操作的时候，内存里会同时存在两个array数组，如果数组内存占用的太大，那么可能会造成频繁GC,所以CopyOnWriteArrayList并不适合大数据量的场景。  

2. 数据一致性  
CopyOnWriteArrayList只能保证数据的最终一致性，不能保证数据的实时一致性——读操作读到的数据只是一份快照。所以如果希望写入的数据可以立刻被读到，那CopyOnWriteArrayList并不适合。

### 1.2 Set

#### CopyOnWriteArraySet  

- 原理

CopyOnWriteArraySet内部引用了一个CopyOnWriteArrayList对象，以“组合”方式，委托CopyOnWriteArrayList对象实现了所有API功能  

- 重要参数

```

    private final CopyOnWriteArrayList<E> al;
    
    public CopyOnWriteArraySet() {
        al = new CopyOnWriteArrayList<E>();
    }
```

- 主要方法

1）add()

```
    // 调用了CopyOnWriteArrayList的addAllAbsent方法。
    return al.addIfAbsent(e);
```

2）其他方法都是委托CopyOnWriteArrayList操作  

### 1.3 BlockingQueue

#### ArrayBlockingQueue 有界阻塞队列

- 原理

1. 队列的容量一旦在构造时指定，后续不能改变；  
2. 插入元素时，在队尾进行；删除元素时，在队首进行；  
3. 队列满时，调用特定方法插入元素会阻塞线程；队列空时，删除元素也会阻塞线程；  
4. 支持公平/非公平策略，默认为非公平策略；
5. 使用一把全局锁，即入队和出队使用同一个ReentrantLock锁  

- 重要参数

```
    // 内部数组
    final Object[] items;

    // 下一个待删除位置的索引: take, poll, peek, remove方法使用
    int takeIndex;

    // 下一个待插入位置的索引: put, offer, add方法使用
    int putIndex;
     
    // 全局重入锁
    final ReentrantLock lock;

    // 非空条件队列：当队列空时，线程在该队列等待获取
    private final Condition notEmpty;

    // 非满条件队列：当队列满时，线程在该队列等待插入
    private final Condition notFull;

    public ArrayBlockingQueue(int capacity) {
        this(capacity, false);
    }
    
    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }
    
    public ArrayBlockingQueue(int capacity, boolean fair, Collection<? extends E> c) {
        this(capacity, fair);
        final ReentrantLock lock = this.lock;
        lock.lock(); // Lock only for visibility, not mutual exclusion
        try {
            int i = 0;
            try {
                for (E e : c) {
                    // 不能有null元素
                    checkNotNull(e);
                    items[i++] = e;
                }
            } catch (ArrayIndexOutOfBoundsException ex) {
                throw new IllegalArgumentException();
            }
            count = i;
            // 如果队列已满，则重置puIndex索引为0
            putIndex = (i == capacity) ? 0 : i;
        } finally {
            lock.unlock();
        }
    }

```

- 主要方法

1）add()

```
    return super.add(e);
    
    if (offer(e))
        return true;
    else
        throw new IllegalStateException("Queue full");
```

2） offer()

```
    // offer(E e)
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        if (count == items.length)
            return false;
        else {
            enqueue(e);
            return true;
        }
    } finally {
        lock.unlock();
    }
    
    // offer(E e, long timeout, TimeUnit unit)
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        // 若队列已满，循环等待被通知，再次检查队列是否非空
        while (count == items.length) {
            // 可等待的时间小于等于零，直接返回失败
            if (nanos <= 0)
                return false;
            // 等待，直到超时, 返回的为剩余可等待时间，相当于每次等待，都会扣除相应已经等待的时间
            nanos = notFull.awaitNanos(nanos);
        }
        enqueue(e);
        return true;
    } finally {
        lock.unlock();
    }
    
    private void enqueue(E x) {
        // assert lock.getHoldCount() == 1;
        // assert items[putIndex] == null;
        final Object[] items = this.items;
        items[putIndex] = x;
        // 队列已满,则重置索引为0
        if (++putIndex == items.length)
            putIndex = 0;
        count++;
        // 唤醒一个notEmpty上的等待线程(可以来队列取元素了)
        notEmpty.signal();
    }
```

3）put()

```
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    // 加锁
    lock.lockInterruptibly();
    try {
        // todo 队列已满; 这里必须用while，防止虚假唤醒
        while (count == items.length)
            // 在notFull队列上等待
            notFull.await();
        // 队列未满, 直接入队
        enqueue(e);
    } finally {
        lock.unlock();
    }
```

4）take()

```
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        // 队列为空, 则线程在notEmpty条件队列等待
        while (count == 0)
            notEmpty.await();
        return dequeue();
    } finally {
        lock.unlock();
    }
    
     private E dequeue() {
        // assert lock.getHoldCount() == 1;
        // assert items[takeIndex] != null;
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        E x = (E) items[takeIndex];
        items[takeIndex] = null;
        // 如果队列已空
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
        // 唤醒一个notFull上的等待线程(可以插入元素到队列了)
        notFull.signal();
        return x;
    }
```

5）poll()

```
    // poll()
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
         // 获得头元素
        return (count == 0) ? null : dequeue();
    } finally {
        lock.unlock();
    }
    
    // poll(long timeout, TimeUnit unit)
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0) {
            if (nanos <= 0)
                return null;
            nanos = notEmpty.awaitNanos(nanos);
        }
        return dequeue();
    } finally {
        lock.unlock();
    }
```

#### LinkedBlockingQueue 有界阻塞队列

- 原理

基本同ArrayBlockingQueue一致, 入队使用一个ReentrantLock锁（putLock），出队使用另一个ReentrantLock锁（takeLock）,不能指定公平/非公平策略（默认都是非公平）

- 重要参数

```
    /** Current number of elements */
    private final AtomicInteger count = new AtomicInteger();

    /** Head of linked list. Invariant: head.item == null */
    transient Node<E> head;

    /** Tail of linked list. Invariant: last.next == null */
    private transient Node<E> last;

    /** Lock held by take, poll, etc */
    private final ReentrantLock takeLock = new ReentrantLock();

    /** Wait queue for waiting takes */
    private final Condition notEmpty = takeLock.newCondition();

    /** Lock held by put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();

    /** Wait queue for waiting puts */
    private final Condition notFull = putLock.newCondition();
    
    // 默认容量 = Integer.MAX_VALUE
    public LinkedBlockingQueue() {
        this(Integer.MAX_VALUE);
    }
    
    public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
        last = head = new Node<E>(null);
    }
    
    public LinkedBlockingQueue(Collection<? extends E> c) {
        this(Integer.MAX_VALUE);
        final ReentrantLock putLock = this.putLock;
        putLock.lock(); // Never contended, but necessary for visibility
        try {
            int n = 0;
            for (E e : c) {
                if (e == null)
                    throw new NullPointerException();
                if (n == capacity)
                    throw new IllegalStateException("Queue full");
                enqueue(new Node<E>(e));
                ++n;
            }
            count.set(n);
        } finally {
            putLock.unlock();
        }
    }
```

- 主要方法

1）offer()

```
    if (e == null) throw new NullPointerException();
    final AtomicInteger count = this.count;
    // 元素个数达到容量
    if (count.get() == capacity)
        return false;
    int c = -1;
    Node<E> node = new Node<E>(e);
    // 获取入队锁
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        if (count.get() < capacity) {
            // 入队
            enqueue(node);
            // c表示入队前的队列元素个数. 赋值给c的是原子本身
            c = count.getAndIncrement();
            // c+1 表示的元素个数. 唤醒一个"入队线程"
            if (c + 1 < capacity)
                notFull.signal();
        }
    } finally {
        putLock.unlock();
    }
    // 队列初始为空, 则唤醒一个“出队线程”
    if (c == 0)
        signalNotEmpty();
    return c >= 0;
    
    
    private void signalNotEmpty() {
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
    }
```

2）put()

``` 
   if (e == null) throw new NullPointerException();
   // Note: convention in all put/take/etc is to preset local var
   // holding count negative to indicate failure unless set.
   int c = -1;
   Node<E> node = new Node<E>(e);
   // 获取入队锁
   final ReentrantLock putLock = this.putLock;
   final AtomicInteger count = this.count;
   putLock.lockInterruptibly();
   try {
       while (count.get() == capacity) {
           notFull.await();
       }
       enqueue(node);
       c = count.getAndIncrement();
       // c+1 元素的个数, 唤醒一个“入队线程"
       if (c + 1 < capacity)
           notFull.signal();
   } finally {
       putLock.unlock();
   }
   if (c == 0)
       signalNotEmpty();
```