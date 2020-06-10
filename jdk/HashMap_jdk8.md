## HashMap之jdk8源码分析

> 包含三种数据结构：
1、数组：hash桶，会进行扩容。
2、链表：hash冲突之后相同hash的元素组成。
3、红黑树：hash冲突之后相同hash元素达到一定阈值后会将链表转为红黑树。

#### 参数定义
```
//初始化默认容器大小
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
//容器size最大的长度
static final int MAXIMUM_CAPACITY = 1 << 30;
//默认扩容阈值系数
static final float DEFAULT_LOAD_FACTOR = 0.75f;
//链表转为红黑树必要条件之一：单链表长度阈值
static final int TREEIFY_THRESHOLD = 8;
//红黑树转为链表时节点个数阈值
static final int UNTREEIFY_THRESHOLD = 6;
//链表转为红黑树必要条件之二：容器内元素数量阈值
static final int MIN_TREEIFY_CAPACITY = 64;
//内部容器，Node数组，可扩容
transient Node<K,V>[] table;

transient Set<Map.Entry<K,V>> entrySet;
//容器元素数量
transient int size;
//容器被修改次数
transient int modCount;
//容器扩容阈值：初始化时等于容器容量大小，后续会设置为capcity*loadFactor
int threshold;
//容器扩容系数：初始化后不可更改
final float loadFactor;
//节点数据结构定义
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
}
```

#### 初始化
>

```
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    //得到大于等于给定容量initialCapacity值的最接近的2的指数倍数值
    this.threshold = tableSizeFor(initialCapacity);
}

//得到2^n，保证结果>=cap
//计算规则：保证cap转化的二进制所有位上都为1，然后+1得到2^n
static final int tableSizeFor(int cap) {
    //防止cap初始值为2^n
    int n = cap - 1;
    //如果可能，保证二进制表示的最高2位为1
    n |= n >>> 1;
    //如果可能，保证二进制表示的最高4位为1
    n |= n >>> 2;
    //如果可能，保证二进制表示的最高8位为1
    n |= n >>> 4;
    //如果可能，保证二进制表示的最高16位为1
    n |= n >>> 8;
    //如果可能，保证二进制表示的最高32位为1
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

#### put
> 采用后插法插入链表；当插入新元素后链表长度大于等于8，容器元素数量大于等于64时，会将链表转为红黑树，提高hash寻址速度。

```
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //如果Node数组尚未初始化，执行resize初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //hash桶的索引位置还未添加元素，创建节点直接添加到指定hash桶上
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        //与hash桶链表头节点位置元素相同
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //hash桶索引位置为红黑树
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            //将新元素采用后插法插入指定位置
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    //创建新节点，并放入链表尾部
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //找到拥有相同key值的元素
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //只有设置了允许替换或者旧值为空的情况下，才设置新值
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    //记录修改次数
    ++modCount;
    //新增加元素后容器元素数量大于阈值，扩容
    if (++size > threshold)
        resize();
    //后置处理
    afterNodeInsertion(evict);
    return null;
}
```

#### 扩容resize
> 容器大小为2^n，每次扩容至2^(n+1)，最大不超过MAXIMUM_CAPACITY。
扩容时会将元素按照规则进行迁移

```
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    //计算新的阈值
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0)
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    //创建新容量大小的数组
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    //将新容器赋值给table，此时table为空，不包含任何元素
    table = newTab;
    //执行迁移：将旧table中的元素迁移到新的table中
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            //hash桶处存在元素
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                //此hash桶只有一个元素，则迁移到新容器指定hash桶（以元素的hash与新容器的容量计算索引）
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                //如果是红黑树节点，按照红黑树的规则拆分
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    //每个hash桶中的元素迁移，按照hash&oldCap的值分成两部分，一部分放到原来索引位置处，一部分放到（原来索引位置+oldCap）处
                    do {
                        next = e.next;
                        //如果元素hash&旧的容量==0，即e.hash的最高位（oldCap=2^n,即在第n索引位（从0开始计数）为0）为0，进行扩容后容器hash时,e.hash&(oldCap<<1-1)，保证定位的桶位置不变，
                        //则位置不变（即原来的旧table的哪个位置，放到新的table中还是那个位置），使用后插法
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        //如果元素e.hash&oldCap不为0，即e.hash的第n索引位（从0开始计数）为1，那么定位到新hash桶的位置e.hash&(newCap-1)，即e.hash&(oldCap<<1-1)，
                        //那么(e.hash&(newCap-1))的二进制形式比(e.hash&(oldCap-1))的二进制表示形式在第n索引位上，比原来的数值多了oldCap大小，那么新的hash桶位置就是j+oldCap
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        //参考上面分析的hash定位
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```
> 关于resize过程中hash桶的迁移，现在指定索引位置j处的hash桶需要迁移到新的table中，jdk1.8在hash桶迁移过程中，元素的hash值是不变的，针对容量的改变，如何保证在resize后元素仍然定位到新的hash桶，这里采用了非常巧妙的设计，下面就来看一看。
假设resize之前table的长度oldCap为16，即2^4,n的大小为4，即二进制表示为0000 0000 0001 0000，有两个个元素的hash值低16位分别为：
hash1 = 0010 0011 0110 1100
hash2 = 0010 0011 0111 1100
那么:
hash1&(oldCap-1) -> (0010 0011 0110 1100)&(0000 0000 0000 1111) -> 0000 0000 0000 1100即table[12]处;
hash2&(oldCap-1) -> (0010 0011 0111 1100)&(0000 0000 0000 1111) -> 0000 0000 0000 1100即与hash1的计算值相同，也是定位到table[12]处。
如果此时table执行resize,newTable的长度newCap=oldCap<<1，即newCap=32,二进制表示0000 0000 0010 0000。
hash1&oldCap -> (0010 0011 0110 1100)&(0000 0000 0001 0000) ->那么此时n位置处为0，
hash2&oldCap -> (0010 0011 0111 1100)&(0000 0000 0001 0000) ->那么此时n位置处为1，
依照迁移规则：
hash1&(newCap-1) -> (0010 0011 0110 1100)&(0000 0000 0001 1111) -> 0000 0000 0000 1100,还是原来hash桶的位置newtable[12]，与oldTable[12]相同，即不用迁移;
hash2&(newCap-1) -> (0010 0011 0111 1100)&(0000 0000 0001 1111) -> 0000 0000 0001 1100,此时hash桶位置因为n位置处为1，
比原来定位的hash桶位置0000 0000 0000 1100多了0000 0000 0001 0000，即oldCap的大小，此时新的桶位置就为newTable[12 + 16]处。

jdk1.8在扩容时相比jdk1.7的改变，因为jdk1.8采用了后插法，而且使用了临时变量，保证每次循环迁移一个桶的位置，在此hash桶迁移过程中不受其他线程的干扰，不会出现jdk1.7的死循环问题。
但是容器并不是线程安全的，会出现并发问题，比如线程1刚好执行resize，创建了newTable，此时newTable为空，没有任何元素，直接赋值给了HashMap的table元素，如果此时有其他线程过来添加元素、取值或者遍历，都会出现问题。另外在keySet/entrySet时，会根据modCount的变化，抛出ConcurrentModificationException异常。
