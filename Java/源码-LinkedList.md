# LinkedList

## 简述

> 1. `LinkedList` 集合底层实现的数据结构为双向链表
> 2. `LinkedList` 集合中元素允许为 null
> 3. `LinkedList` 允许存入重复的数据
> 4. `LinkedList` 中元素存放顺序为存入顺序。
> 5. `LinkedList` 是非线程安全的，如果想保证线程安全的前提下操作 `LinkedList`，可以使用 `List list = Collections.synchronizedList(new LinkedList(...));` 来生成一个线程安全的 `LinkedList`

## 继承关系

![](..\images\Java\Collection\LinkedList继承.png)

实现了List接口和Deque接口的双端链表

```java
extends AbstractSequentialList<E> implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```

## Node

内部类

```java
private static class Node<E> {
    E item;			// 当前节点元素
    Node<E> next;	// 下一个节点索引
    Node<E> prev;	// 上一个节点索引

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

## 成员变量

```java
transient int size = 0;		// 节点个数
transient Node<E> first;	// 链表的第一个节点
transient Node<E> last;		// 链表的最后一个节点
```

## 构造函数

```java
// 空链表, first = last = null
public LinkedList() {
}

public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
```

1. 检查索引值是否合法，不合法将抛出角标越界异常
2. 保存 index 位置的节点，和 index-1 位置的节点，对于单链表熟悉的同学一定清楚对于链表的增删操作都需要两个指针变量完成。
3. 将参数集合转化为数组，循环将数组中的元素封装为节点添加到链表中。
4. 更新链表长度并返回添加 true 表示添加成功。

## 方法

### 增加节点

#### add

```java
// 链表尾部
public boolean add(E e) {
    linkLast(e);
    return true;
}

void linkLast(E e) {
    // 保存之前链表尾部元素
	final Node<E> l = last;
    // 新建新节点,prev指向之前链表的last
	final Node<E> newNode = new Node<>(l, e, null);
    // last索引指向新建的节点
	last = newNode;
    // 如果之前链表为空，那么first也指向新节点
	if (l == null)
		first = newNode;
	else
		l.next = newNode;	// 将之前末节点的nect指向新节点
	size++;
	modCount++;
}

public void add(int index, E element) {
	checkPositionIndex(index);

	if (index == size)
		linkLast(element);
	else
		linkBefore(element, node(index));
}

void linkBefore(E e, Node<E> succ) {
	// assert succ != null;
	final Node<E> pred = succ.prev;
	final Node<E> newNode = new Node<>(pred, e, succ);
	succ.prev = newNode;
	if (pred == null)
		first = newNode;
	else
		pred.next = newNode;
	size++;
	modCount++;
}
```

| ![](D:\Work\IDEA\maoyz.github.io\images\Java\Collection\LinkedList-add(E).gif) |
| :----------------------------------------------------------: |
| ![](D:\Work\IDEA\maoyz.github.io\images\Java\Collection\LinkedList-add(index, E).gif) |
| ![](..\images\Java\Collection\LinkedList-add(index, E)-1.gif) |



#### addAll

```java
public boolean addAll(Collection<? extends E> c) {
	return addAll(size, c);
}

public boolean addAll(int index, Collection<? extends E> c) {
    // 检验索引  0 <= index <= size
	checkPositionIndex(index);
	// 转数组
	Object[] a = c.toArray();
    // 新数组长度
	int numNew = a.length;
	if (numNew == 0)
		return false;
	// 保存index当前的节点succ，当前节点的上一个节点pred
	Node<E> pred, succ;
    // 链表尾部插入
	if (index == size) {
		succ = null;
		pred = last;
	} else {
		succ = node(index);
		pred = succ.prev;
	}
	// 遍历数组
	for (Object o : a) {
		@SuppressWarnings("unchecked") E e = (E) o;
		Node<E> newNode = new Node<>(pred, e, null);
        // 表示链表中没有元素，元素赋值给第一个节点
		if (pred == null)
			first = newNode;
		else
			pred.next = newNode;
		pred = newNode;
	}
	// index位置的元素为null
	if (succ == null) {
		last = pred;
	} else {
		pred.next = succ;
		succ.prev = pred;
	}

	size += numNew;
	modCount++;
	return true;
}

Node<E> node(int index) {
    // assert isElementIndex(index);
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

#### addFirst

```java
public void addFirst(E e) {
    linkFirst(e);
}

private void linkFirst(E e) {
	final Node<E> f = first;
	final Node<E> newNode = new Node<>(null, e, f);
	first = newNode;
	if (f == null)
		last = newNode;
	else
		f.prev = newNode;
	size++;
	modCount++;
}
```



#### addLast

```java
public void addLast(E e) {
    linkLast(e);
}
```

### 查询节点

#### get

```java
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}
```

|  ![](..\images\Java\Collection\LinkedList-get(index).gif)  |
| :--------------------------------------------------------: |
| ![](..\images\Java\Collection\LinkedList-get(index)-1.gif) |



#### getFirst

```java
public E getFirst() {
	final Node<E> f = first;
	if (f == null)
		throw new NoSuchElementException();
	return f.item;
}
```

#### getLast

```java
public E getLast() {
	final Node<E> l = last;
	if (l == null)
		throw new NoSuchElementException();
	return l.item;
}
```

### 修改节点

#### set

```java
public E set(int index, E element) {
    checkElementIndex(index);
    Node<E> x = node(index);
    E oldVal = x.item;
    x.item = element;
    return oldVal;
}
```

### 删除节点

#### remove

```java
public E remove() {
    return removeFirst();
}

public E remove(int index) {
	checkElementIndex(index);
	return unlink(node(index));
}

public boolean remove(Object o) {
	if (o == null) {
		for (Node<E> x = first; x != null; x = x.next) {
			if (x.item == null) {
				unlink(x);
				return true;
			}
		}
	} else {
		for (Node<E> x = first; x != null; x = x.next) {
			if (o.equals(x.item)) {
				unlink(x);
				return true;
			}
		}
	}
	return false;
}

E unlink(Node<E> x) {
	// assert x != null;
	final E element = x.item;
	final Node<E> next = x.next;
	final Node<E> prev = x.prev;

	if (prev == null) {
		first = next;
	} else {
		prev.next = next;
		x.prev = null;
	}

	if (next == null) {
		last = prev;
	} else {
		next.prev = prev;
		x.next = null;
	}

	x.item = null;
	size--;
	modCount++;
	return element;
}
```
![](..\images\Java\Collection\LinkedList-remove(E).gif)

#### removeFirst

```java
public E removeFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}

private E unlinkFirst(Node<E> f) {
	// assert f == first && f != null;
	final E element = f.item;
	final Node<E> next = f.next;
	f.item = null;
	f.next = null; // help GC
	first = next;
	if (next == null)
		last = null;
	else
		next.prev = null;
	size--;
	modCount++;
	return element;
}
```

#### removeLast

```java
public E removeLast() {
    final Node<E> l = last;
    if (l == null)
        throw new NoSuchElementException();
    return unlinkLast(l);
}

private E unlinkLast(Node<E> l) {
	// assert l == last && l != null;
	final E element = l.item;
	final Node<E> prev = l.prev;
	l.item = null;
	l.prev = null; // help GC
	last = prev;
	if (prev == null)
		first = null;
	else
		prev.next = null;
	size--;
	modCount++;
	return element;
}
```



参考文章：

https://github.com/Snailclimb/JavaGuide/blob/master/docs/java/collection/LinkedList.md

https://github.com/Snailclimb/JavaGuide/blob/master/docs/java/collection/LinkedList.md

https://www.cnblogs.com/xdecode/p/9321848.html