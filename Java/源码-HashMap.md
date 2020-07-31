# HashMap

![](..\images\Java\Map\Map体系.png)



Node

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
```

TreeNode

```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
	TreeNode<K,V> parent;  // red-black tree links
	TreeNode<K,V> left;
	TreeNode<K,V> right;
	TreeNode<K,V> prev;    // needed to unlink next upon deletion
	boolean red;
	TreeNode(int hash, K key, V val, Node<K,V> next) {
		super(hash, key, val, next);
	}
}
```



```java
// 默认初始容量16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
// 最大容量
static final int MAXIMUM_CAPACITY = 1 << 30;
// 默认加载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;
// 当桶(bucket)上的结点数大于这个值时会转成红黑树
static final int TREEIFY_THRESHOLD = 8;
// 当桶(bucket)上的结点数小于这个值时树转链表
static final int UNTREEIFY_THRESHOLD = 6;
// 桶中结构转化为红黑树对应的table的最小大小
static final int MIN_TREEIFY_CAPACITY = 64;
// 存放元素的数组，2的幂次方
transient Node<K,V>[] table;
// 存放具体元素的集
transient Set<Map.Entry<K,V>> entrySet;
// 存放元素的个数，不是数组的长度
transient int size;

transient int modCount;
// 临界值，当实际大小(容量*加载因子)超过临界值时，会进行扩容
int threshold;
// 加载因子
final float loadFactor;
```

- 加载因子loadFactor

  **loadFactor太大导致查找元素效率低，太小导致数组的利用率低，存放的数据会很分散。loadFactor的默认值为0.75f是官方给出的一个比较好的临界值**。

  给定的默认容量为 16，负载因子为 0.75。Map 在使用过程中不断的往里面存放数据，当数量达到了 16 * 0.75 = 12 就需要将当前 16 的容量进行扩容，而扩容这个过程涉及到 rehash、复制数据等操作，所以非常消耗性能。

- 临界值threshold

  threshold = capacity \* loadFactor，当Size>=threshold的时候，那么就要考虑对数组的扩增了，也就是说，这个的意思就是 衡量数组是否需要扩增的一个标准。



<img src="..\images\Java\Map\HashMap-put逻辑.jpg" style="zoom: 67%;" />

参考文档：

https://github.com/Snailclimb/JavaGuide/blob/master/docs/java/collection/HashMap.md