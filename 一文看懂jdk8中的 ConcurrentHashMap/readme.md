# 一文看懂jdk8中的 ConcurrentHashMap

> 相信大家在日常开发中都用过 HashMap，HashMap 在并发扩容过程中，在 jdk7 中的实现可能会形成环形链表从而引发死循环，在jdk8中的实现又可能造成数据覆盖的问题。因此不论是 jdk7 还是 jdk8，HashMap 都是线程不安全的，为了解决线程安全问题，对Java 发展影响深远的 **Doug Lea** 编写了 ConcurrentHashMap 供开发者使用。
>
> 本文就 ConcurrentHashMap 的实现原理做初步探讨。



## HashMap存在的问题

任何技术的诞生都是有其独特的诞生背景的，HashMap 诞生于分治思想，而 ConcurrentHashMap 则是为了解决 HashMap 中的线程安全问题而生，接下来我们就一起看一下 HashMap 中存在的线程安全问题。

* 先看下 jdk7 中扩容方法的实现

  ```java
  void resize(int newCapacity) {
      Entry[] oldTable = table;
      int oldCapacity = oldTable.length;
      if (oldCapacity == MAXIMUM_CAPACITY) {
          threshold = Integer.MAX_VALUE;
          return;
      }
      Entry[] newTable = new Entry[newCapacity];
      // 最终会进入transfer方法
      transfer(newTable, initHashSeedAsNeeded(newCapacity));
      table = newTable;
  	threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
  }
  
  
  ```

  `resize` 方法会调用 `transfer`方法

  ```java
  void transfer(Entry[] newTable, boolean rehash) {
      int newCapacity = newTable.length;
      for (Entry<K,V> e : table) {
          while(null != e) {
              Entry<K,V> next = e.next;
              if (rehash) {
                  e.hash = null == e.key ? 0 : hash(e.key);
              }
              int i = indexFor(e.hash, newCapacity);
              e.next = newTable[i];
              newTable[i] = e;
              e = next;
          }
      }
  }
  ```

  

  如图所示，该map在插入第四个元素时会触发扩容

  ![ByXdBT.md.png](https://s1.ax1x.com/2020/11/03/ByXdBT.md.png)

  

  

  假设有两个线程同时执行到 `transfer` 方法，线程2执行到 `Entry<K,V> next = e.next; `之后cpu时间片耗尽，此时变量e指向节点a，变量next指向节点b。线程1完成了扩容，变量引用关系如图所示：

  ![ByLvsf.md.png](https://s1.ax1x.com/2020/11/03/ByLvsf.md.png)

  线程2开始执行，此时对于线程2中保存的变量引用关系如图所示：

  ![ByLjQP.md.png](https://s1.ax1x.com/2020/11/03/ByLjQP.md.png)

  执行后，变量e指向节点b，因为e不是null，则继续执行循环体，执行后的引用关系。

  ![ByLOzt.md.png](https://s1.ax1x.com/2020/11/03/ByLOzt.md.png)

  变量e又重新指回节点a，只能继续执行循环体，这里仔细分析下：
   1、执行完`Entry<K,V> next = e.next;`，目前节点a没有next，所以变量next指向null；
   2、`e.next = newTable[i];` 其中 newTable[i] 指向节点 b，那就是把 a 的 next 指向了节点 b，这样 a 和 b 就相互引用了，形成了一个环；
   3、`newTable[i] = e` 把节点a放到了数组i位置；
   4、`e = next;` 把变量e赋值为 null，因为第一步中变量 next 就是指向 null；

  所以最终的引用关系是这样的：

![ByLLRI.md.png](https://s1.ax1x.com/2020/11/03/ByLLRI.md.png)

​		节点a和b互相引用，形成了一个环，当在数组该位置get寻找对应的key时，就发生了死循环。

​		另外，如果线程2把 newTable 设置成到内部的 table，节点 c 的数据就丢了，看来还有数据遗失的问题。

​	

* jdk8中的数据覆盖问题

  ```java
  final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                     boolean evict) {
          Node<K,V>[] tab; Node<K,V> p; int n, i;
          if ((tab = table) == null || (n = tab.length) == 0)
              n = (tab = resize()).length;
          if ((p = tab[i = (n - 1) & hash]) == null)
               // 如果没有hash碰撞则直接插入元素
              tab[i] = newNode(hash, key, value, null);
          else {
             //...
          }
          ++modCount;
          if (++size > threshold)
              resize();
          afterNodeInsertion(evict);
          return null;
      }
  ```

  1、假设两个线程1、2都在进行 put 操作，并且hash函数计算出的插入下标是相同的。

  2、当线程1执行完第六行代码后由于时间片耗尽导致被挂起，而线程2得到时间片后在该下标处插入了元素，完成了正常的插入。

  3、然后线程1获得时间片，由于之前已经进行了 hash 碰撞的判断，所有此时不会再进行判断，而是直接进行插入。

  4、这就导致了线程2插入的数据被线程1覆盖了，从而线程不安全。





## ConcurrentHashMap实现原理

ConcurrentHashMap 在 jdk7 升级j到 dk8之 后有较大的改动，jdk7 中主要采用 `Segment` 分段锁的思想，`Segment` 继承自`ReentrantLock` 类，依次来保证线程安全。限于篇幅原因，本文只讨论 jdk8 中的 ConcurrentHashMap 实现原理。有兴趣的同学可以自行研究 jdk7 中的实现。

jdk8 中的 ConcurrentHashMap 数据结构同 jdk8 中的 HashMap 数据结构一样，都是 **数组+链表+红黑树**。摒弃了 jdk7 中的分段锁设计，使用了 `Node` + `CAS` + `Synchronized` 来保证线程安全。



### 重要成员变量

```markdown
table：默认为 null，初始化发生在第一次插入操作，默认大小为16的数组，用来存储Node节点数据，扩容时大小总是2的幂次方。

nextTable：默认为 null，扩容时新生成的数组，其大小为原数组的两倍。

sizeCtl：默认为0，用来控制table的初始化和扩容操作，具体应用在后续会体现出来。 
	-1 代表 table 正在初始化 
	-N 表示有 N-1 个线程正在进行扩容操作 
	其余情况： 
		1、如果 table 未初始化，表示 table 需要初始化的大小。 
		2、如果 table 初始化完成，表示 table 的容量，默认是 table 大小的0.75倍，用这个公式算 0.75（n - (n >>> 2)）。

Node：保存 key，value 及 key 的 hash 值的数据结构。其中 value 和 next 都用 **volatile** 修饰，保证并发的可见性。

ForwardingNode：一个特殊的 Node 节点，hash 值为-1，其中存储 nextTable 的引用。只有 table 发生扩容的时候，ForwardingNode 才会发挥作用，作为一个占位符放在 table 中表示当前节点为 null 或则已经被移动。
```

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    // volatile 修饰的变量可以保证线程可见性，同时也可以禁止指令重排序
    // 有关 volatile 原理此处不展开，volatile 的实现原理可自行上网查阅
    volatile V val;
    volatile Node<K,V> next;
    //...
}
```



### 重要方法实现原理

在探究方法实现之前，我们先认识一下 `Unsafe` 和 CAS 思想，ConcurrentHashMap 中大量用到 `Unsafe`  类和 CAS 思想。

**Unsafe**

```markdown
Unsafe 是 jdk 提供的一个直接访问操作系统资源的工具类（底层c++实现），它可以直接分配内存，内存复制，copy，提供 cpu 级别的 CAS 乐观锁等操作。它的目的是为了增强java语言直接操作底层资源的能力。使用Unsafe类最主要的原因是避免使用高速缓冲区中的过期数据。

为了方便理解，举个栗子。类 User 有一个成员变量 name。我们new了一个对象 User 后，就知道了它在内存中的起始值 ,而成员变量 name 在对象中的位置偏移是固定的。这样通过这个起始值和这个偏移量就能够定位到 name 在内存中的具体位置。
Unsafe 提供了相应的方法获取静态成员变量，成员变量偏移量的方法，所以我们可以使用 Unsafe 直接更新内存中 name 的值。
```

**CAS**

```markdown
CAS 译为 Compare And Swap，它是乐观锁的一种实现。假设内存值为 v，预期值为 e，想要更新成得值为 u，当且仅当内存值v等于预期值e时，才将v更新为u。CAS 操作不需要加锁，所以比起加锁的方式 CAS 效率更高。
```



------



接下来我们来看看具体的方法实现。



#### size 方法

`size` 方法用于统计 map 中的元素个数，通过源码我们发现 `size` 核心是  `sumCount`  方法，其中变量 baseCount  的值是记录完成元素插入并且成功更新到 baseCount  上的元素个数，CounterCell 数组是记录完成元素插入但是在 CAS 修改 baseCount 失败时的元素个数，因此 baseCount + CounterCell 数组记录的总数是 map 中的元素个数。

这样设计的原因是高并发情况下大量的 CAS 修改 baseCount 的值是失败的，为了节约性能，CAS 更新 baseCount 失败之后用 CounterCell 数组记录下来，CounterCell 初始化数组长度为2，高并发情况可以扩容，每个数组节点分别记录落在当前数组的记录数，使用的是 CAS 去操作 value++，最后将所有节点的 value 求和，并加上 baseCount 的值，即为 map 元素个数。

```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}
```

```java
final long sumCount() {
    // CounterCell 数组是修改 baseCount 失败的线程放入的数量
    CounterCell[] as = counterCells; CounterCell a;
    // baseCount 是修改 baseCount 成功的线程放入的数量
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```



-----

#### put 方法

`put` 方法是向map中存入元素，本质上调用了 `putVal`，该方法非常核心的方法之一，读者可结合笔者添加的注释加以理解

```java
 final V putVal(K key, V value, boolean onlyIfAbsent) {	
     if (key == null || value == null) throw new NullPointerException();
     int hash = spread(key.hashCode());
     int binCount = 0;
     for (Node<K,V>[] tab = table;;) {
         Node<K,V> f; int n, i, fh;
         // 如果 tab 未初始化，先初始化 tab，此处是懒加载的思想
         if (tab == null || (n = tab.length) == 0)
             tab = initTable();
         // 如果计算出来的 tab 下标位置上没有其他元素，用 CAS 操作建立引用
         else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
             if (casTabAt(tab, i, null,
                          new Node<K,V>(hash, key, value, null)))
                 break;                   // no lock when adding to empty bin
         }
         // 如果发现当前节点的哈希值是 MOVED，则说明正处于扩容状态中，当前线程加入扩容大军，帮助扩容
         else if ((fh = f.hash) == MOVED)
             tab = helpTransfer(tab, f);
         else {
             V oldVal = null;
             // 哈希冲突，锁住当前节点
             synchronized (f) {
                 if (tabAt(tab, i) == f) {
                     // fh>=0说明是链表，遍历寻找
                     if (fh >= 0) {
                         binCount = 1;
                         for (Node<K,V> e = f;; ++binCount) {
                             K ek;
                             // 发现已经存在相同的 key，根据 onlyIfAbsent 判断是否覆盖
                             if (e.hash == hash &&
                                 ((ek = e.key) == key ||
                                  (ek != null && key.equals(ek)))) {
                                 oldVal = e.val;
                                 if (!onlyIfAbsent)
                                     e.val = value;
                                 break;
                             }
                             // 继续向下遍历
                             Node<K,V> pred = e;
                             if ((e = e.next) == null) {
                                 pred.next = new Node<K,V>(hash, key,
                                                           value, null);
                                 break;
                             }
                         }
                     }
                     // 如果是红黑树，调用 putTreeVal 方法，遍历树，此处逻辑不详细展开
                     else if (f instanceof TreeBin) {
                         Node<K,V> p;
                         binCount = 2;
                         if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                               value)) != null) {
                             oldVal = p.val;
                             if (!onlyIfAbsent)
                                 p.val = value;
                         }
                     }
                 }
             }
             if (binCount != 0) {
                 // 此时 binCount 表示一个节点对应链表的长度，到达8就转换成红黑树
                 if (binCount >= TREEIFY_THRESHOLD)
                     treeifyBin(tab, i);
                 // 返回旧值
                 if (oldVal != null)
                     return oldVal;
                 break;
             }
         }
     }
     // 判断是否需要扩容
     addCount(1L, binCount);
     return null;
 }
```



小结 `put` ：

1. 先校验一个 k 和 v 都不可为空。
2. 判断 table 是否为空，如果为空就进入初始化阶段。
3. 如果发现插入位置的 bucket 为空，直接把键值对插入到这个桶中作为头节点。
4. 如果这个要插入的桶中的 hash 值为 - 1，也就是 MOVED 状态（也就是这个节点是 forwordingNode ），那就是说明有线程正在进行扩容操作，那么当前线程就进入协助扩容阶段。
5. 如果这个节点是一个链表节点，根据 key 匹配结果去决定是插入还是覆盖，插入是用尾插法。如果这个节点是一个红黑树节点，那就需要按照树的插入规则进行插入。
6. 插入结束之后判断该链表节点个数是否到达8，如果是就把链表转化为红黑树存储。
7. put 结束之后，需要给 map 已存储的数量 +1，在 addCount 方法中判断是否需要扩容。

```markdown
总结一下扩容条件：

1. 元素个数达到扩容阈值。

2. 调用 putAll 方法，但目前容量不足以存放所有元素时。

3. 某条链表长度达到8，但数组长度却小于64时，该逻辑在 treeifyBin 方法中。
```



-----



通读了`putVal`之后，我们比较关注其中一些方法：

* `tabAt` 方法是通过 `Unsafe` 类根据偏移量直接从内存中获取数据，避免了从高速缓冲区获得了过期数据

* `casTabAt` 方法主要通过 `Unsafe` 类直接操作内存，通过比较交换赋值，该操作不用加锁，所以可以提高操作效率

  ```java
  @SuppressWarnings("unchecked")
  static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
      return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
  }
  static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                      Node<K,V> c, Node<K,V> v) {
      return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
  }
  ```

* `initTable` 方法初始化 map 中的底层数组

  ```java
  private final Node<K,V>[] initTable() {
      Node<K,V>[] tab; int sc;
      while ((tab = table) == null || tab.length == 0) {
          // 如果一个线程发现 sizeCtl < 0 ，意味着另外的线程执行 CAS 操作成功，当前线程只需要让出 CPU 时间片
          if ((sc = sizeCtl) < 0)
              // Thread.yield() 方法是让当前线程主动放弃 CPU 的执行权，线程回到就绪状态
              Thread.yield(); // lost initialization race; just spin
          // 通过 CAS 设置 sizeCtl 为 -1
          else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
              try {
                  if ((tab = table) == null || tab.length == 0) {
                      int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                      @SuppressWarnings("unchecked")
                      Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                      table = tab = nt;
                      //sc = 0.75 * n，此处的sc即为扩容的阈值
                      sc = n - (n >>> 2);
                  }
              } finally {
                  //将sizeCtl设置成扩容阈值
                  sizeCtl = sc;
              }
              break;
          }
      }
      return tab;
  }
  ```

  

* `addCount` 方法用于判断是否需要扩容

  ```java
  private final void addCount(long x, int check) {
      CounterCell[] as; long b, s;
      if ((as = counterCells) != null ||
          // 已经放入到 map 中但是更新 baseCount 失败，放到 CounterCell 数组中
          !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
          CounterCell a; long v; int m;
          boolean uncontended = true;
          // 如果 CounterCell 数组是空（尚未出现并发）
          // 如果随机取余一个数组位置为空 或者
          // 修改这个槽位的变量失败（出现并发了）
          if (as == null || (m = as.length - 1) < 0 ||
              (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
              !(uncontended =
                U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
              // 执行 fullAddCount 方法，将更新 baseCount 失败的元素个数保存到CounterCell 数组中
              fullAddCount(x, uncontended);
              return;
          }
          if (check <= 1)
              return;
  		// s 是总节点数量；
          s = sumCount();
      }
      // 扩容逻辑
      if (check >= 0) {
          Node<K,V>[] tab, nt; int n, sc;
          while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                 (n = tab.length) < MAXIMUM_CAPACITY) {
              // 根据 length 得到一个标识
              int rs = resizeStamp(n);
              // 如果正在扩容
              if (sc < 0) {
                  // 如果 sc 的低 16 位不等于 标识符（校验异常 sizeCtl 变化了）
                  // 如果 sc == 标识符 + 1 （扩容结束了，不再有线程进行扩容）（默认第一个线程设置 sc ==rs 左移 16位 + 2，当第一个线程结束扩容了，就会将 sc 减一。这个时候，sc 就等于 rs + 1）
                  // 如果 sc == 标识符 + 65535（帮助线程数已经达到最大）
                  // 如果 nextTable == null（结束扩容了）
                  // 如果 transferIndex <= 0 (转移状态变化了)
                  // 结束循环
                  if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                      sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                      transferIndex <= 0)
                      break;
                  // 如果可以帮助扩容，那么将 sc 加 1. 表示多了一个线程在帮助扩容
                  if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                      transfer(tab, nt);
              }
              // 如果不在扩容，将 sc 更新：标识符左移 16 位 然后 + 2. 也就是变成一个负数。高 16 位是标识符，低 16位初始是 2.
              else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                           (rs << RESIZE_STAMP_SHIFT) + 2))
  				// 更新 sizeCtl 为负数后，开始扩容。
                  transfer(tab, null);
              s = sumCount();
          }
      }
  }
  ```





----

#### get 方法

`get` 方法逻辑比较简单清晰

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        // 判断链表头结点是否匹配 key
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // eh < 0 ，如果是红黑树，去红黑树上查找，如果是 ForwardingNode ，则会跳到扩容后的 map 上寻找
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        // 遍历链表查找
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```



------

#### helpTransfer  和 transfer 方法

`helpTransfer`  方法在判断完扩容状态后，本质上还是调用了 `transfer`

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    // 当前节点是 ForwardingNode 并且已经初始化完成新的数组，帮助扩容
    if (tab != null && (f instanceof  ForwardingNode 并且已经初始化完成新的数组，帮助扩容) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            // 扩容完成的标志
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            // 开始帮助扩容
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```



`transfer` 方法逻辑比较复杂，请读者结合注释和配图耐心理解

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // CPU 核心数大于1，每个线程负责迁移16个 bucket
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    // 初始化扩容后的数组
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
    // 通过for自循环处理每个槽位中的链表元素，默认advace为真，通过CAS设置transferIndex属性值，并初始化i和bound值，i指当前处理的槽位序号，bound指需要处理的槽位边界，先处理槽位15的节点；
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        while (advance) {
            int nextIndex, nextBound;
            // 所有节点都有线程负责或者已经扩容完成
            if (--i >= bound || finishing)
                advance = false;
            // 将 i 值置-1
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                //确定当前线程每次分配的待迁移桶的范围为[bound, nextIndex)
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        // 当前线程自己的活已经做完或所有线程的活都已做完，第二与第三个条件应该是下面让"i = n"后，再次进入循环时要做的边界检查。
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            // 扩容完成的后续工作
            if (finishing) {
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            // 采用CAS算法更新SizeCtl,设置 finishing 和 advance 标志
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        // 当前节点为 null，直接标记成 forwordingNode 
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // 当前节点已经迁移过
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            // 锁住头结点
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    //这里 ln，hn 是高低链表的思想，详细过程下图中会有说明
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
                    // 红黑树也是高低链表的思想，详细过程下图中会有说明
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







* **多线程开始扩容**



[![BcM5rt.jpg](https://s1.ax1x.com/2020/11/04/BcM5rt.jpg)](https://imgchr.com/i/BcM5rt)





* **lastrun节点**



[![BcM2Pe.jpg](https://s1.ax1x.com/2020/11/04/BcM2Pe.jpg)](https://imgchr.com/i/BcM2Pe)







* **链表迁移**



[![BcMW2d.jpg](https://s1.ax1x.com/2020/11/04/BcMW2d.jpg)](https://imgchr.com/i/BcMW2d)











* **红黑树迁移**



[![BcMR8H.jpg](https://s1.ax1x.com/2020/11/04/BcMR8H.jpg)](https://imgchr.com/i/BcMR8H)







* **迁移过程中get和put的操作的处理**

[![BcMc5D.jpg](https://s1.ax1x.com/2020/11/04/BcMc5D.jpg)](https://imgchr.com/i/BcMc5D)





* **并发迁移**

[![BcMfxA.jpg](https://s1.ax1x.com/2020/11/04/BcMfxA.jpg)](https://imgchr.com/i/BcMfxA)





* **迁移完成**

[![BclH3Q.jpg](https://s1.ax1x.com/2020/11/04/BclH3Q.jpg)](https://imgchr.com/i/BclH3Q)

小结 `transfer`:

1. 根据 CPU 核心数确定每个线程负责的桶数，默认每个线程16个桶
2. 创建新数组，长度是原来数组的两倍
3. 分配好当前线程负责的桶区域 [bound, nextIndex)
4. 并发迁移，根据链表和红黑树执行不同迁移策略
5. 迁移完成，设置新的数组和新的扩容阈值

----

至此，笔者已经把 ConcurrentHashMap 几个重要的方法实现介绍完了。剩下的如 `remove` 、`replace` 等方法实现都大同小异，读者可自行研究。



## 总结

通过以上对 ConcurrentHashMap 的初步探讨，相信读者也会和笔者一样惊叹于 **Doug Lea** 大师的编程水平和技巧。ConcurrentHashMap  类在 jdk8 中源码多达6300 行，其中运用了大量的多线程与高并发的编程技术，如 volatile、synchronized、CAS、Unsafe、Thread 类相关 API，以及许多精巧的算法，如 ConcurrentHashMap 底层数组的长度是2的幂次方以便用位运算计算元素下标 ，同时也方便计算扩容后的元素下标，还有令笔者惊叹的高低链表迁移操作，诸如此类、不再赘述。

感谢 **Doug Lea** 对于 Java 发展做出的贡献，也希望我们可以向大师学习，磨炼自己的编程水平。借用笔者很喜欢的一个程序员大佬的一句话，学习是一条令人时而欣喜若狂、时而郁郁寡欢的道路。共勉！

最后，笔者水平有限，文中若有错误欢迎指出。



**参考文章**：

<https://www.jianshu.com/p/1e9cf0ac07f4>

<https://blog.csdn.net/programmer_at/article/details/79715177>

<https://blog.csdn.net/zzu_seu/article/details/106698150>
