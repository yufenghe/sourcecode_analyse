## PriorityBlockingQueue源码分析

> 使用最小堆来实现的可扩容有界可阻塞的优先级队列。
主要数据结构为：
- DEFAULT_INITIAL_CAPACITY，队列初始化默认容量大小。
- MAX_ARRAY_SIZE，队列容量最大值。
- queue，Object数组，存放队列元素。
- size，队列元素数量。
- comparator，比较器，因为最小堆，队列对象排序比较。
- lock，ReentrantLock独占锁
- notEmpty，等待非空条件
- allocationSpinLock，volatile修饰，cas设值，值为0或1，用于扩容标识。
- q，PriorityQueue队列，序列化和反序列化使用，平常为null。

PriorityBlockingQueue类图结构如下：
![](../assets/jdk/PriorityBlockingQueue_class_inher.png)
