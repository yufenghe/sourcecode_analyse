## ConcurrentHashMap的源码分析（jdk8）

> 之前分洗了jdk7版本的ConcurrentHashMap，接下来看下jdk8的ConcurrentHashMap做了哪些修改，其中包括数据结构和非阻塞实现方式。

先看以下jdk8下ConcurrentHashMap的结构图：
![](../assets/jdk/concurrenthashmap_jdk8_structure.png)

ConcurrentHashMap基于使用锁分离技术，使用cas和synchronized实现非阻塞更新，以数组+链表+树的数据结构实现的多线程并发容器。

### Node结构
> Node中val和next使用volatile修饰，保证内存可见性。

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

      //查询节点
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

  //特殊类型的Node，标识正在迁移，可以索引到新的table，以便在还没有完全迁移完时定位到昂前hash桶时数据的操作
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
### ConcurrentHashMap定义

```java
public class ConcurrentHashMap<K, V> extends AbstractMap<K, V>
        implements ConcurrentMap<K, V>, Serializable {
    //最大容量
    private static final int MAXIMUM_CAPACITY = 1 << 30;
    //默认容量
    private static final int DEFAULT_CAPACITY = 16;
    //最大数组长度
    static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
    //默认并发度：jdk8的并发度是不断变化的，扩容会导致增加
    private static final int DEFAULT_CONCURRENCY_LEVEL = 16;
    //加载因子
    private static final float LOAD_FACTOR = 0.75f;
    //链表转为树结构的阈值：一个hash桶中hash冲突的数目大于此值时，将链表转为红黑树，加快hash冲突下的查找速度
    static final int TREEIFY_THRESHOLD = 8;
    //将树转为链表结构的阈值：一个hash桶中冲突数目小于此值时，将红黑树转为链表
    static final int UNTREEIFY_THRESHOLD = 6;
    //当table数组长度小于此值时，不会将链表转为红黑树，所以链表转化为红黑树有两个条件：hash桶中冲突数目大于TREEIFY_THRESHOLD且容器中的数据大于此值
    static final int MIN_TREEIFY_CAPACITY = 64;
    //每个线程负责迁移Node数组的长度，该值最起码要大于等于DEFAULT_CAPACITY
    private static final int MIN_TRANSFER_STRIDE = 16;
    //用于每次扩容生成戳的数
    private static int RESIZE_STAMP_BITS = 16;
    //最大的扩容线程的数量，如果按照默认值的话，那就是2^16-1个线程进行扩容
    private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
    //移位量，把生成戳移位后保存在sizeCtl中当做扩容线程计数的基数，相反方向移位后能够反解出生成戳
    private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

    // 下面几个是特殊的节点的hash值，正常节点的hash值在hash函数中都处理过了，不会出现负数的情况，特殊节点在各自的实现类中有特殊的遍历方法
    // ForwardingNode的hash值，ForwardingNode是一种临时节点，在扩进行中才会出现，并且它不存储实际的数据
    // 如果旧数组的一个hash桶中全部的节点都迁移到新数组中，旧数组就在这个hash桶中放置一个ForwardingNode
    // 读操作或者迭代读时碰到ForwardingNode时，将操作转发到扩容后的新的table数组上去执行，写操作碰见它时，则尝试帮助扩容
    static final int MOVED     = -1;
    //红黑树根节点的hash值
    static final int TREEBIN   = -2;
    static final int RESERVED  = -3;
    static final int HASH_BITS = 0x7fffffff;

    //CPU数，用于扩容时计算每个线程负责迁移的槽数
    static final int NCPU = Runtime.getRuntime().availableProcessors();

    //ConcurrentHashMap的实际容器，Node数组
    transient volatile Node<K,V>[] table;
    //扩容用，正常情况下为null
    private transient volatile Node<K,V>[] nextTable;

    /**
     * 多线程之间，以volatile的方式读取sizeCtl属性，来判断ConcurrentHashMap当前所处的状态。通过cas设置sizeCtl属性，告知其他线程ConcurrentHashMap的状态变更。
     * sizeCtl为负数是表示正在初始化或者扩容
     * sizeCtl=0：未初始化
     * sizeCtl>0：未初始化状态，表示初始容量
     * sizeCtl=-1：标记作用，告诉其他线程正在初始化
     * sizeCtl=-(1 + nThreads)：表示有nThreads个线程正在进行扩容操作
     * sizeCtl=0.75n：正常状态，表示扩容阈值
     */
    private transient volatile int sizeCtl;
    //下一个执行transfer任务的下标，有transferIndex和bound决定了线程处理的槽数，transfer时下标index从length-1往0的递减的方向变化
    private transient volatile int transferIndex;

    // CAS自旋锁标志位，用于初始化，或者counterCells扩容时
    private transient volatile int cellsBusy;    
    // 用于高并发的计数单元，如果初始化了这些计数单元，那么跟table数组一样，长度必须是2^n的形式
    private transient volatile CounterCell[] counterCells;

    public ConcurrentHashMap() {
    }

    //指定初始容量的构造函数
    public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        this.sizeCtl = cap;
    }

    public ConcurrentHashMap(int initialCapacity, float loadFactor) {
        this(initialCapacity, loadFactor, 1);
    }

    public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        //设置的初始容量应该大于等于并发度
        if (initialCapacity < concurrencyLevel)
            initialCapacity = concurrencyLevel;
        //计算实际的容量，必须是2^n，该值为大于等于size的最小2^n
        long size = (long)(1.0 + (long)initialCapacity / loadFactor);
        int cap = (size >= (long)MAXIMUM_CAPACITY) ?
            MAXIMUM_CAPACITY : tableSizeFor((int)size);
        //sizeCtl初始为实际容量值
        this.sizeCtl = cap;
    }
}
#### tableSizeFor
用户计算出大于等于给定容量的2^n，即2^n>=cap
```java
  private static final int tableSizeFor(int c) {
        //避免次数为2^n
        int n = c - 1;
        //计算后n的最高前2位为1
        n |= n >>> 1;
        //计算后n的最高前4位都为1
        n |= n >>> 2;
        //计算后n的最高前8位都为1
        n |= n >>> 4;
        //计算后n的最高前16位都为1
        n |= n >>> 8;
        //计算后n的最高前32位都为1
        n |= n >>> 16;
        //现在计算出的n每位数字都为1，执行n+1操作，就可以保证最高位为1，其他位为0，即实现了2^n
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```
>tableSizeFor采用位移方式计算，将cap的所有位数先变成1，在执行+1操作，就变成了2^n，此值正好>=设置的容量。

```
### put使用
```java
  public V put(K key, V value) {
      return putVal(key, value, false);
  }

  final V putVal(K key, V value, boolean onlyIfAbsent) {
      //key和value都不能为空
      if (key == null || value == null) throw new NullPointerException();
      //计算hash
      int hash = spread(key.hashCode());
      //用于hash桶节点计数
      int binCount = 0;
      for (Node<K,V>[] tab = table;;) {
          Node<K,V> f; int n, i, fh;
          //如果为空则执行初始化
          if (tab == null || (n = tab.length) == 0)
              tab = initTable();
          //如果索引到的hash桶为空，则cas设置当前节点到hash桶，结束循环
          else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
              if (casTabAt(tab, i, null,
                           new Node<K,V>(hash, key, value, null)))
                  break;                   // no lock when adding to empty bin
          }
          //如果当前hash桶正在执行transfer操作，即扩容，则当前线程协助transfer
          else if ((fh = f.hash) == MOVED)
              tab = helpTransfer(tab, f);
          else {
              V oldVal = null;
              //当前hash槽加锁
              synchronized (f) {
                  //判断hash槽头结点是否变化，如果没变继续执行，否则重新执行for循环
                  if (tabAt(tab, i) == f) {
                      //头结点hash>=0，表示已经初始化完成，存在其他hash冲突的数据，将新节点放入链表
                      if (fh >= 0) {
                          binCount = 1;
                          for (Node<K,V> e = f;; ++binCount) {
                              K ek;
                              if (e.hash == hash &&
                                  ((ek = e.key) == key ||
                                   (ek != null && key.equals(ek)))) {
                                  oldVal = e.val;
                                  if (!onlyIfAbsent)
                                      e.val = value;
                                  break;
                              }
                              Node<K,V> pred = e;
                              if ((e = e.next) == null) {
                                  pred.next = new Node<K,V>(hash, key,
                                                            value, null);
                                  break;
                              }
                          }
                      }
                      //判断头结点是否为红黑树节点，将新节点放入红黑树
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
              //如果hash槽中为链表，判断当前槽中节点数是否超过转化为红黑树的阈值
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
```
#### initTable
>执行容器的初始化操作。
```java
  private final Node<K,V>[] initTable() {
      Node<K,V>[] tab; int sc;
      while ((tab = table) == null || tab.length == 0) {
          //如果size<0表示有其他线程在初始化或者扩容，当前线程让出CPU挂起
          if ((sc = sizeCtl) < 0)
              Thread.yield();
          //cas设置sizeCtrl值为-1，标识容器正在初始化，cas成功则标识获得了锁，其他失败线程则等待获取初始化权限的线程初始化完毕，退出当前循环
          else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
              try {
                  //再次检查
                  if ((tab = table) == null || tab.length == 0) {
                      //默认容量
                      int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                      @SuppressWarnings("unchecked")
                      Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                      table = tab = nt;
                      //计算sc新值，即初始化后容器的阈值：最后结果为0.75n
                      //n>>>2为n/4=0.25n
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
#### helpTransfer
```java
  final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
      Node<K,V>[] nextTab; int sc;
      //因为f的hash为MOVED，即为ForwardingNode类型，标识正在迁移
      if (tab != null && (f instanceof ForwardingNode) &&
          (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
          //最高位为1，其余为tab.length最高位1前面的0的个数，即32-index(top1)
          int rs = resizeStamp(tab.length);
          //判断是否仍在迁移操作，当前sc为-(1+nThreads)
          while (nextTab == nextTable && table == tab &&
                 (sc = sizeCtl) < 0) {
              //如果迁移完成，或者负责迁移的线程数量达到最大值，则终止循环
              if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                  sc == rs + MAX_RESIZERS || transferIndex <= 0)
                  break;
              //sizeCtl表示当前迁移的线程数，当前线程设置此值+1，成功，则进入迁移操作
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
### transfer
> 执行Node数组的扩容和元素的迁移操作。

```java
  private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
      int n = tab.length, stride;
      //计算每个线程迁移的槽数，如果为多核机器，判断当前table的槽数/8/NCPU是否小于最少迁移槽数16
      if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
          stride = MIN_TRANSFER_STRIDE;
      //迁移table尚未初始化
      if (nextTab == null) {
          try {
              @SuppressWarnings("unchecked")
              //双倍容量初始化
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
      //迁移标识节点，通过next连接到nextTable，在迁移过程中不影响该位置出元素的读取
      ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
      //迁移使用
      boolean advance = true;
      boolean finishing = false;
      for (int i = 0, bound = 0;;) {
          Node<K,V> f; int fh;
          //刚进入for循环时执行此循环，用于设置tranferIndex，获得本线程要执行迁移的区间[transferIndex, transferIndex + stride]，通过更新transferIndex来获得多线程协同
          while (advance) {
              int nextIndex, nextBound;
              if (--i >= bound || finishing)
                  advance = false;
              else if ((nextIndex = transferIndex) <= 0) {
                  i = -1;
                  advance = false;
              }
              //设置transferIndex，数值从n到0倒序变化，每个线程迁移的table槽数为stride，通过设置transferIndex，其他线程就可以根据此值设置自己的迁移方案
              else if (U.compareAndSwapInt
                       (this, TRANSFERINDEX, nextIndex,
                        nextBound = (nextIndex > stride ?
                                     nextIndex - stride : 0))) {
                  bound = nextBound;
                  i = nextIndex - 1;
                  advance = false;
              }
          }
          //i如果小于0
          if (i < 0 || i >= n || i + n >= nextn) {
              int sc;
              //迁移结束，收尾工作
              if (finishing) {
                  //正常状态为空
                  nextTable = null;
                  //指向迁移完毕table
                  table = nextTab;
                  //新的阈值：2*n-0.5n=0.75*(2*n)
                  sizeCtl = (n << 1) - (n >>> 1);
                  return;
              }
              //本线程执行完分配的迁移工作，执行sc-1
              if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                  //判断是否为执行迁移操作的最后一个线程，因为执行迁移时sizeCtl=resizeStamp(n) << RESIZE_STAMP_SHIFT+2
                  if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                      return;
                  finishing = advance = true;
                  i = n; // recheck before commit
              }
          }
          //当前位置尚未初始化，即不存在元素，不需要迁移
          else if ((f = tabAt(tab, i)) == null)
              advance = casTabAt(tab, i, null, fwd);
          //其他线程正在迁执行此位置的迁移，hash标识为MOVED
          else if ((fh = f.hash) == MOVED)
              advance = true; // already processed
          else {
              //加锁操作：锁住当前位置的Node
              synchronized (f) {
                  //因为多线程，再次检查节点是否变化
                  if (tabAt(tab, i) == f) {
                      Node<K,V> ln, hn;
                      //hash>=0，为链表节点，否则为红黑树节点
                      if (fh >= 0) {
                          int runBit = fh & n;
                          Node<K,V> lastRun = f;
                          //会将原来链表节分成两部分，
                          for (Node<K,V> p = f.next; p != null; p = p.next) {
                              int b = p.hash & n;
                              if (b != runBit) {
                                  runBit = b;
                                  lastRun = p;
                              }
                          }
                          //hash&n为0设置低水位索引位置，否则设置到高水位索引位置，初始化ln，hn
                          if (runBit == 0) {
                              ln = lastRun;
                              hn = null;
                          }
                          else {
                              hn = lastRun;
                              ln = null;
                          }
                          //具体拆分原链表到新链表的操作：i，i+n,采用头插法
                          for (Node<K,V> p = f; p != lastRun; p = p.next) {
                              int ph = p.hash; K pk = p.key; V pv = p.val;
                              //hash&n==0设置到低水位，即原来相同的位置i，否则设置到原来的位置i+n处
                              if ((ph & n) == 0)
                                  ln = new Node<K,V>(ph, pk, pv, ln);
                              else
                                  hn = new Node<K,V>(ph, pk, pv, hn);
                          }
                          //cas设置low，high处的链表头结点
                          setTabAt(nextTab, i, ln);
                          setTabAt(nextTab, i + n, hn);
                          //设置原来Node数组i位置的迁移标识
                          setTabAt(tab, i, fwd);
                          advance = true;
                      }
                      //如果为红黑树的情况
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
