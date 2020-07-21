# ArrayList

## 特点

1. ArrayList 底层是一个动态扩容的数组结构  
2. 允许存放多个null 元素  
3. 允许存放重复数据，存储顺序按照元素的添加顺序  
4. ArrayList并不是一个线程安全的集合。如果集合的增删操作需要保证线程的安全性，可以考虑使用 CopyOnWriteArrayList 或者使用 collections.synchronizedList(List l)函数返回一个线程安全的ArrayList类.  

## 继承关系

```
public class ArrayList<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```


一. 实现 `RandomAccess` 标记接口  

1.表明List提供了随机访问功能，也就是通过下标获取元素对象的功能。之所以是标记接口，是该类本来就具有某项能力，使用接口对其进行标签化，便于其他的类对其进行识别（instanceof）

2.ArrayList数组实现，本身就有通过下标随机访问任意元素的功能。那么需要细节上注意的就是随机下标访问和顺序下标访问（LinkedList）的不同了。也就是为什么LinkedList最好不要使用循环遍历，而是用迭代器遍历的原因。

3.实现RandomAccess同时意味着一些算法可以通过类型判断进行一些针对性优化，例子有Collections的shuffle方法  

简单说就是，如果实现RandomAccess接口就下标遍历，反之迭代器遍历


二. 实现了Cloneable, java.io.Serializable意味着可以被克隆和序列化

为什么ArrayList的elementData是用transient修饰的?

elementData不总是满的，每次都序列化，会浪费时间和空间, 重写了writeObject 保证序列化的时候虽然不序列化全部, 但是有的元素都序列化
所以说不是不序列化, 而是不全部序列化。


```java
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{
        // Write out element count, and any hidden stuff
        int expectedModCount = modCount;
        s.defaultWriteObject();

        // Write out size as capacity for behavioural compatibility with clone()
        s.writeInt(size);

        // Write out all elements in the proper order.
        for (int i=0; i<size; i++) {
            s.writeObject(elementData[i]);
        }

        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }

   
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        elementData = EMPTY_ELEMENTDATA;

        // Read in size, and any hidden stuff
        s.defaultReadObject();

        // Read in capacity
        s.readInt(); // ignored

        if (size > 0) {
            // be like clone(), allocate array based upon size not capacity
            int capacity = calculateCapacity(elementData, size);
            SharedSecrets.getJavaOISAccess().checkArray(s, Object[].class, capacity);
            ensureCapacityInternal(size);

            Object[] a = elementData;
            // Read in all elements in the proper order.
            for (int i=0; i<size; i++) {
                a[i] = s.readObject();
            }
        }
    }
```

## 成员变量

```java
    // 无参构造时，默认10
    private static final int DEFAULT_CAPACITY = 10;

    // 共享的空的数组实例，当使用 ArrayList(0) 或者 ArrayList(Collection<? extends E> c), 并且 c.size() = 0 的时候讲 elementData 数组讲指向这个实例对象。
    private static final Object[] EMPTY_ELEMENTDATA = {};

    // 另一个共享空数组实例，再第一次 add 元素的时候将使用它来判断数组大小是否设置为 DEFAULT_CAPACITY
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    transient Object[] elementData;

    private int size;
    
    // 构造一个初始容量为10的空列表, 原因: 第一次添加元素肯定走进 if 判断中 minCapacity 将被赋值为 10
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
    
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+ initialCapacity);
        }
    }
```

## 方法

1. add()

```java
    public boolean add(E e) {
        // 检查当前底层数组容量，如果容量不够则进行扩容
        ensureCapacityInternal(size + 1);
        elementData[size++] = e;
        return true;
    }
    
    // 1. 检查当前底层数组容量，如果容量不够则进行扩容
    private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }
    
    // 2 如果是无参构造方法构造的的集合，第一次添加元素的时候会满足这个条件 minCapacity 将会被赋值为 10
    private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }

    // 3. 将 size + 1 或 10 传入 ensureExplicitCapacity 进行扩容判断
    private void ensureExplicitCapacity(int minCapacity) {
        // 操作数加 1 用于保证并发访问 
        modCount++;

        // 如果当前数组的长度比添加元素后的长度要小则进行扩容 
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    
    // 4. 扩容
    private void grow(int minCapacity) {
        // 获取当前 elementData 的大小，也就是 List 中当前的容量
        int oldCapacity = elementData.length;
        // 新容量为当前容量的 1.5 倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        // 如果扩大1.5倍后仍旧比 minCapacity 小那么直接等于 minCapacity
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        //如果新数组大小比  MAX_ARRAY_SIZE 就需要进一步比较 minCapacity 和 MAX_ARRAY_SIZE 的大小
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // 使用 Arrays.copyOf 构建一个长度为 newCapacity 新数组 并将 elementData 指向新数组
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
    
    // 5. 比较 minCapacity 与 Integer.MAX_VALUE-8(减少出错的几率) 的大小如果大则放弃-8的设定，设置为 Integer.MAX_VALUE 
    private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
    }

```

2. addAll()

```java
    public boolean addAll(Collection<? extends E> c) {
        // 调用 c.toArray 将集合转化数组
        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
    }
    
    private void rangeCheck(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

```

3. toArray()

```java
    public <T> T[] toArray(T[] a) {
        // 1. 如果 a.length < size 即当前集合元素的个数与参数 a 数组元素的大小的时候将和 toArray() 一样返回一个新的数组
        if (a.length < size)
            return (T[]) Arrays.copyOf(elementData, size, a.getClass());
        // 2. 将不会产生新的数组直接将集合中的元素调用 System.arraycopy 方法将元素复制到参数数组中，返回 a
        System.arraycopy(elementData, 0, a, 0, size);
        // 3. a.length > size 也不会产生新的数组,但是值得注意的是 a[size] = null; 改变了原数组中 index = size 位置的元素，被重新设置为 null 了
        if (a.length > size)
            a[size] = null;
        return a;
    }
```




参考文章: [https://juejin.im/post/5ab548f75188257ddb0f8fa2#heading-32](https://note.youdao.com/)