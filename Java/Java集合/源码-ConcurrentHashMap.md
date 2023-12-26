# ConcurrentHashMap

<img src="image/Java/Map/java8_concurrenthashmap.png" style="zoom:80%;" />

其中抛弃了原有的 Segment 分段锁，直接采用transient volatile HashEntry<K,V>[] table保存数据，而采用了 `CAS + synchronized` 来保证并发安全性，底层采用数组+链表+红黑树的存储结构。

```java
static class Node<K,V> implements Map.Entry<K,V> {
	final int hash;
	final K key;
	volatile V val;
	volatile Node<K,V> next;

	Node(int hash, K key, V val, Node<K,V> next) {
		this.hash = hash;
		this.key = key;
		this.val = val;
		this.next = next;
	}

	public final K getKey()       { return key; }
	public final V getValue()     { return val; }
	public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }
	public final String toString(){ return key + "=" + val; }
	public final V setValue(V value) {
		throw new UnsupportedOperationException();
	}

	public final boolean equals(Object o) {
		Object k, v, u; Map.Entry<?,?> e;
		return ((o instanceof Map.Entry) &&
				(k = (e = (Map.Entry<?,?>)o).getKey()) != null &&
				(v = e.getValue()) != null &&
				(k == key || k.equals(key)) &&
				(v == (u = val) || v.equals(u)));
	}

	/**
	 * Virtualized support for map.get(); overridden in subclasses.
	 */
	Node<K,V> find(int h, Object k) {
		Node<K,V> e = this;
		if (k != null) {
			do {
				K ek;
				if (e.hash == h &&
					((ek = e.key) == k || (ek != null && k.equals(ek))))
					return e;
			} while ((e = e.next) != null);
		}
		return null;
	}
}
```

其中的 `val`  和 `next` 都用了 volatile 修饰，保证了可见性。

```java
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        super(MOVED, null, null, null);
        this.nextTable = tab;
    }

    Node<K,V> find(int h, Object k) {
        // loop to avoid arbitrarily deep recursion on forwarding nodes
        outer: for (Node<K,V>[] tab = nextTable;;) {
            Node<K,V> e; int n;
            if (k == null || tab == null || (n = tab.length) == 0 ||
                (e = tabAt(tab, (n - 1) & h)) == null)
                return null;
            for (;;) {
                int eh; K ek;
                if ((eh = e.hash) == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
                if (eh < 0) {
                    if (e instanceof ForwardingNode) {
                        tab = ((ForwardingNode<K,V>)e).nextTable;
                        continue outer;
                    }
                    else
                        return e.find(h, k);
                }
                if ((e = e.next) == null)
                    return null;
            }
        }
    }
}
```



```java
// 默认为null，初始化发生在第一次插入操作，默认大小为16的数组，用来存储Node节点数据，扩容时大小总是2的幂次方。
transient volatile Node<K,V>[] table;
// 默认为null，扩容时新生成的数组，其大小为原数组的两倍。
private transient volatile Node<K,V>[] nextTable;


private transient volatile long baseCount;

// 默认为0，用来控制table的初始化和扩容操作
// -1 代表table正在初始化
// -N 表示有N-1个线程正在进行扩容操作
// 其余情况：
// 1、如果table未初始化，表示table需要初始化的大小。
// 2、如果table初始化完成，表示table的容量，默认是table大小的0.75倍，居然用这个公式算0.75（n - (n >>> 2)）。
private transient volatile int sizeCtl;


private transient volatile int transferIndex;

private transient volatile int cellsBusy;
```

```java
public V put(K key, V value) {
	return putVal(key, value, false);
}

/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // key和val都不能为Null
	if (key == null || value == null) throw new NullPointerException();
	int hash = spread(key.hashCode());  // 根据hashcode计算hash值
	int binCount = 0;
    // 遍历
	for (Node<K,V>[] tab = table;;) {
		Node<K,V> f; int n, i, fh; // f:当前key定位的Node
        // 如果桶为空，进行初始化
		if (tab == null || (n = tab.length) == 0)
			tab = initTable();
		else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
			if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
				break;                   // no lock when adding to empty bin
		}
		else if ((fh = f.hash) == MOVED)
			tab = helpTransfer(tab, f);
		else {
			V oldVal = null;
			synchronized (f) {
				if (tabAt(tab, i) == f) {
					if (fh >= 0) {
						binCount = 1;
						for (Node<K,V> e = f;; ++binCount) {
							K ek;
							if (e.hash == hash && ((ek = e.key) == key || (ek != null && key.equals(ek)))) {
								oldVal = e.val;
								if (!onlyIfAbsent)
									e.val = value;
								break;
							}
							Node<K,V> pred = e;
							if ((e = e.next) == null) {
								pred.next = new Node<K,V>(hash, key, value, null);
								break;
							}
						}
					}
					else if (f instanceof TreeBin) {
						Node<K,V> p;
						binCount = 2;
						if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {
							oldVal = p.val;
							if (!onlyIfAbsent)
								p.val = value;
						}
					}
				}
			}
			if (binCount != 0) {
				if (binCount >= TREEIFY_THRESHOLD)
					treeifyBin(tab, i);
				if (oldVal != null)
					return oldVal;
				break;
			}
		}
	}
	addCount(1L, binCount);
	return null;
}

// 初始桶结构
private final Node<K,V>[] initTable() {
	Node<K,V>[] tab; int sc;
	while ((tab = table) == null || tab.length == 0) {
        // 说明其他线程CAS成功，正在进行初始化
		if ((sc = sizeCtl) < 0)
			Thread.yield(); // lost initialization race; just spin
		else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
			try {
				if ((tab = table) == null || tab.length == 0) {
					int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
					
					Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
					table = tab = nt;
					sc = n - (n >>> 2);
				}
			} finally {
				sizeCtl = sc;
			}
			break;
		}
	}
	return tab;
}
```

1. 根据key计算hashcode
2. 判断是否需要初始化
3. 根据当前key定位的Node，如果为空表示当前位置可以写入数据，利用CAS
4. 如果当前位置的 `hashcode == MOVED == -1`,则需要进行扩容
5. 如果都不满足，利用synchronized锁写入数据
6. 数量大于TREEIFY_THRESHOLD（8）则转化为红黑树

```java
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}

final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
	Node<K,V>[] nextTab; int sc;
	if (tab != null && (f instanceof ForwardingNode) && (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
		int rs = resizeStamp(tab.length);
		while (nextTab == nextTable && table == tab && (sc = sizeCtl) < 0) {
			if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
				sc == rs + MAX_RESIZERS || transferIndex <= 0)
				break;
			if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
				transfer(tab, nextTab);
				break;
			}
		}
		return nextTab;
	}
	return table;
}

private final void addCount(long x, int check) {
	CounterCell[] as; long b, s;
	if ((as = counterCells) != null ||
		!U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
		CounterCell a; long v; int m;
		boolean uncontended = true;
		if (as == null || (m = as.length - 1) < 0 ||
			(a = as[ThreadLocalRandom.getProbe() & m]) == null ||
			!(uncontended =  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
			fullAddCount(x, uncontended);
			return;
		}
		if (check <= 1)
			return;
		s = sumCount();
	}
	if (check >= 0) {
		Node<K,V>[] tab, nt; int n, sc;
		while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
			   (n = tab.length) < MAXIMUM_CAPACITY) {
			int rs = resizeStamp(n);
			if (sc < 0) {
				if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
					sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
					transferIndex <= 0)
					break;
				if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
					transfer(tab, nt);
			}
			else if (U.compareAndSwapInt(this, SIZECTL, sc,
										 (rs << RESIZE_STAMP_SHIFT) + 2))
				transfer(tab, null);
			s = sumCount();
		}
	}
}

private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
	int n = tab.length, stride;
	if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
		stride = MIN_TRANSFER_STRIDE; // subdivide range
	if (nextTab == null) {            // initiating
		try {
			@SuppressWarnings("unchecked")
			Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
			nextTab = nt;
		} catch (Throwable ex) {      // try to cope with OOME
			sizeCtl = Integer.MAX_VALUE;
			return;
		}
		nextTable = nextTab;
		transferIndex = n;
	}
	int nextn = nextTab.length;
	ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
	boolean advance = true;
	boolean finishing = false; // to ensure sweep before committing nextTab
	for (int i = 0, bound = 0;;) {
		Node<K,V> f; int fh;
		while (advance) {
			int nextIndex, nextBound;
			if (--i >= bound || finishing)
				advance = false;
			else if ((nextIndex = transferIndex) <= 0) {
				i = -1;
				advance = false;
			}
			else if (U.compareAndSwapInt
					 (this, TRANSFERINDEX, nextIndex,
					  nextBound = (nextIndex > stride ?
								   nextIndex - stride : 0))) {
				bound = nextBound;
				i = nextIndex - 1;
				advance = false;
			}
		}
		if (i < 0 || i >= n || i + n >= nextn) {
			int sc;
			if (finishing) {
				nextTable = null;
				table = nextTab;
				sizeCtl = (n << 1) - (n >>> 1);
				return;
			}
			if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
				if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
					return;
				finishing = advance = true;
				i = n; // recheck before commit
			}
		}
		else if ((f = tabAt(tab, i)) == null)
			advance = casTabAt(tab, i, null, fwd);
		else if ((fh = f.hash) == MOVED)
			advance = true; // already processed
		else {
			synchronized (f) {
				if (tabAt(tab, i) == f) {
					Node<K,V> ln, hn;
					if (fh >= 0) {
						int runBit = fh & n;
						Node<K,V> lastRun = f;
						for (Node<K,V> p = f.next; p != null; p = p.next) {
							int b = p.hash & n;
							if (b != runBit) {
								runBit = b;
								lastRun = p;
							}
						}
						if (runBit == 0) {
							ln = lastRun;
							hn = null;
						}
						else {
							hn = lastRun;
							ln = null;
						}
						for (Node<K,V> p = f; p != lastRun; p = p.next) {
							int ph = p.hash; K pk = p.key; V pv = p.val;
							if ((ph & n) == 0)
								ln = new Node<K,V>(ph, pk, pv, ln);
							else
								hn = new Node<K,V>(ph, pk, pv, hn);
						}
						setTabAt(nextTab, i, ln);
						setTabAt(nextTab, i + n, hn);
						setTabAt(tab, i, fwd);
						advance = true;
					}
					else if (f instanceof TreeBin) {
						TreeBin<K,V> t = (TreeBin<K,V>)f;
						TreeNode<K,V> lo = null, loTail = null;
						TreeNode<K,V> hi = null, hiTail = null;
						int lc = 0, hc = 0;
						for (Node<K,V> e = t.first; e != null; e = e.next) {
							int h = e.hash;
							TreeNode<K,V> p = new TreeNode<K,V>
								(h, e.key, e.val, null, null);
							if ((h & n) == 0) {
								if ((p.prev = loTail) == null)
									lo = p;
								else
									loTail.next = p;
								loTail = p;
								++lc;
							}
							else {
								if ((p.prev = hiTail) == null)
									hi = p;
								else
									hiTail.next = p;
								hiTail = p;
								++hc;
							}
						}
						ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
							(hc != 0) ? new TreeBin<K,V>(lo) : t;
						hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
							(lc != 0) ? new TreeBin<K,V>(hi) : t;
						setTabAt(nextTab, i, ln);
						setTabAt(nextTab, i + n, hn);
						setTabAt(tab, i, fwd);
						advance = true;
					}
				}
			}
		}
	}
}
```

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 && (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```



采用红黑树之后可以保证查询效率（`O(logn)`），甚至取消了 ReentrantLock 改为了 synchronized，这样可以看出在新版的 JDK 中对 synchronized 优化是很到位的。

参考文章：

https://juejin.im/post/6844903641866846222#heading-13

