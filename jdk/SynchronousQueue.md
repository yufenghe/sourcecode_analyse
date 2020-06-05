## SynchronousQueue源码分析

> 类似管道，生产-消费者模式，是一个没有容量的阻塞队列，线程添加元素，需要等待取元素的线程配对成功，才能继续操作，存与取是成对存在的。
内部数据结构为：
- NCPUS，机器CPU数目。
- maxTimedSpins，带有超时等待时的自旋次数。
- maxUntimedSpins，非超时等待的自旋次数。
- spinForTimeoutThreshold，线程从自旋到阻塞等待的自旋时间阈值。
- transferer，TransferStack（后进先出，使用栈实现非公平队列，即后来的线程先配对）或TransferQueue（先进先出，使用队列实现公平队列，即等待时间最久的线程先配对）。

SynchronousQueue的类图结构如下：
![](../assets/jdk/SynchronousQueue_class_inher.png)

#### 初始化
> 默认初始化为非公平队列。

```
public SynchronousQueue() {
    this(false);
}

public SynchronousQueue(boolean fair) {
    transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
}
```

#### add
> 添加元素，调用offer实现，但是添加失败会抛出异常。

```
public boolean add(E e) {
    if (offer(e))
        return true;
    else
        throw new IllegalStateException("Queue full");
}
```

#### offer
> 元素为null抛出异常；然后调用transferer执行配对；可超时。

```
public boolean offer(E e) {
    //元素不能为null
    if (e == null) throw new NullPointerException();
    return transferer.transfer(e, true, 0) != null;
}
```

#### put
> 元素为null抛出异常；与offer不同的是：不可超时，添加失败，清除中断标识，抛出中断异常。

```
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    if (transferer.transfer(e, false, 0) == null) {
        Thread.interrupted();
        throw new InterruptedException();
    }
}
```

#### TransferQueue.transfer
> 生产者(put之类请求)的transfer请求e不为null，消费者（poll之类请求）e为null。
公平模式的请求配对，是尾部判断，头部匹配，可以这么理解：
如果新来的请求与尾部的请求相同，而当前模式下为公平模式，先来者还没有完成匹对，新来者就只能排队等待，而且最终完成匹配的是头结点（等待时间最长的节点），依照公平原则，当然是先匹配。
如果新来的请求与尾部的请求不同，说明前面有配对需求，那么直接从头部开始匹配。
这种方式，很好的处理了公平性原则。

```
E transfer(E e, boolean timed, long nanos) {
    QNode s = null;
    //判断是否数据节点
    //如果传入的为null，表示是消费者
    boolean isData = (e != null);

    //自旋，无限循环
    for (;;) {
        QNode t = tail;
        QNode h = head;
        //尚未初始化队列，返回继续自旋
        if (t == null || h == null)   
            continue;               

        // h==t表示队列为空
        // t.isData == isData表示尾部等待的线程与当前线程是同一种模式的节点（都是生产节点或者都是消费节点）
        if (h == t || t.isData == isData) {
            //获得尾结点的下一节点
            QNode tn = t.next;
            //发生了不一致读，即尾结点发生了变动（因为多线程环境，tail为volatile修饰，多线程保证可见性），重新自旋
            if (t != tail)
                continue;
            //如果尾结点的下一节点不为null（添加了相同模式的新节点，即已经执行过t.casNext，新节点已经添加到尾结点的next），则更新尾结点，重新自旋
            if (tn != null) {
                advanceTail(t, tn);
                continue;
            }

            //执行到此步：t==tail，且tail.next==null
            //设置了超时，达到超时条件，因为队列尾结点与当前节点是相同模式节点，配对失败，直接返回null
            if (timed && nanos <= 0)        // can't wait
                return null;

            //执行到此步：不允许超时或者等待时间>0

            //如果新节点尚未初始化，初始化新节点
            if (s == null)
                s = new QNode(e, isData);

            //将新节点设置为尾结点的next
            //如果添加失败（此种情况表示存在多线程竞争，有其他线程已经添加了节点），重新自旋
            if (!t.casNext(null, s))
                continue;

            //将s节点添加成功到队列中，更新尾结点，始终保持tail指向最后一个节点
            advanceTail(t, s);
            Object x = awaitFulfill(s, e, timed, nanos);
            //判断当前线程节点是否发生中断被取消等待配对（tryCancel时会将QNode.item设置为当前QNode）
            if (x == s) {
                //如果s不为尾结点，则直接移除，否则设置cleanMe变量为s的前驱                   
                clean(t, s);
                return null;
            }
            //x!=s，表示被其他请求节点完成了配对请求，并被唤醒，此时x为请求节点的item值
            //完成配对后，会设置head.next=head，释放旧的head节点，判断此节点是否已经unlinked
            if (!s.isOffList()) {          
                //如果s的前驱t为head节点，则更新head指向s
                advanceHead(t, s);       
                //如果当前节点是请求数据节点，即consumer
                if (x != null)            
                    s.item = s;//cancelled节点就是此处理方式
                //将节点的线程置为null
                s.waiter = null;
            }
            //返回配对结果
            return (x != null) ? (E)x : e;

        } else { //当前线程配对请求时，与尾结点的isData不同，即一个是数据节点一个是请求节点，正好配对                          
            QNode m = h.next;               
            //如果尾节点发生变化或者队列为空，或者头结点发生变化，发生不一致读，存在其他线程竞争，重新循环
            if (t != tail || m == null || h != head)
                continue;                  

            //取第一个请求的线程节点
            Object x = m.item;
            //如果正常匹配，最终当前请求节点与第一个节点匹配成功，并且将请求节点的item值设置到第一个节点的item中去
            if (isData == (x != null) ||    // 判断两个节点模式是否相同（配对相同后），如果为false继续
                x == m ||                   // 判断m节点是否中断取消，如果 为false继续
                !m.casItem(x, e)) {         // cas将请求节点的item值设置到第一个节点的item中，如果cas设值失败，说明有其他线程匹配第一个节点成功，重新设置head指向，然后下一次循环
                //第一个数据节点已经完成了配对，重新定位head节点，然后重试
                advanceHead(h, m);
                continue;
            }

            //新节点与第一个节点完成配对和设置item操作，修正head指向
            advanceHead(h, m);         
            //唤醒第一个节点等待的线程
            LockSupport.unpark(m.waiter);
            //返回匹配的数据
            return (x != null) ? (E)x : e;
        }
    }
}
```
#### TransferQueue.clean
> 清理cancelled节点，清理原则：
1、如果节点为尾结点，则设置cleanMe节点，等待后续清理。
2、如果节点不是尾结点，则直接清理cancelled节点。

```
void clean(QNode pred, QNode s) {
    //清空当前QNode的等待线程值
    s.waiter = null;

    //t==pred为true
    //初始情况下pred.next==s为true，因为clean之前设置了t.casNext(null,s)，所以此处t不一定是倒数第二个节点，但肯定是s的前继节点
    //只有新加入的节点为t的next节点才执行，此处是pred.next==s为true，除非s节点已经从队列中移除
    while (pred.next == s) {
        QNode h = head;
        QNode hn = h.next;  
        //定位head节点
        //先判断head的next节点是否已经被取消，如果被取消，则更新head节点，重新循环
        if (hn != null && hn.isCancelled()) {
            advanceHead(h, hn);
            continue;
        }

        QNode t = tail;      
        //如果tail==head，表示队列空，直接返回
        if (t == h)
            return;
        QNode tn = t.next;
        //如果尾结点发生了变更，重新循环，保证t指向尾结点
        if (t != tail)
            continue;
        //尾结点插入了新的节点，更新尾结点，始终保证tail指向尾结点
        if (tn != null) {
            advanceTail(t, tn);
            continue;
        }

        //中间节点中断取消等待后，执行unlink操作
        //到此步，tail为队列的尾结点，而t也指向tail，如果s!=t为true，说明s不是最后一个节点
        if (s != t) {
            QNode sn = s.next;
            //1、如果s==s.next，表示s节点已经unlink，则clean结束；
            //2、s!=s.next，即s尚未unlink，则执行s节点的unlink操作，即设置pred的next指向，即删除s节点,clean流程结束
            if (sn == s || pred.casNext(s, sn))
                return;
        }

        //执行到此处，说明s节点为队列最后一个节点，而根据SynchronousQueue的规则，最后一个节点不能直接unlink，需要使用cleanMe变量记录它的前驱节点pred
        QNode dp = cleanMe;

        //s节点取消之前，有取消的尾结点，执行之前取消节点的unlink操作
        if (dp != null) {
            QNode d = dp.next;
            QNode dn;
            if (d == null ||               // ① d为null，即dp为尾结点，即之前cancelled节点已经unlink，dp没有关联节点了，变为无效节点，执行casCleanMe or
                d == dp ||                 // ② d不为null且dp.next==d，表示dp自身执行了unlink，变为无效节点，执行casCleanMe or
                !d.isCancelled() ||        // ③ d不为null，没有执行unlink，且d 没有被cacelled，表示dp节点有效，执行casCleanMe or
                (d != t &&                 // ④ d被取消了，并且d不是尾结点 and d.next不为null，表示d不是最后一个节点 and d.next!=d，表示d未执行unlink
                 (dn = d.next) != null &&   
                 dn != d &&                
                 dp.casNext(d, dn)))       // 因为d执行了取消，并且上面判断了d不是最后一个节点，则更新dp.next，即把d节点从队列中移除了
                casCleanMe(dp, null);      //满足①、②、③、④中任何一个条件，dp的任务已经完成，更新cleanMe变量值
            //如果cleanMe==pred，表示已经设置了s节点的前驱为cleanMe，直接返回
            if (dp == pred)
                return;      
        } else if (casCleanMe(null, pred))//设置s的前驱为cleanMe
            return;         
    }
}
```

#### advanceTail
> 更新尾结点。

```
void advanceTail(QNode t, QNode nt) {
    if (tail == t)
        UNSAFE.compareAndSwapObject(this, tailOffset, t, nt);
}
```
#### TransferQueue.awaitFulfill
> 当前节点如果与尾结点是同一类型（isData相同，同为生产者或者消费者），按照以下方式处理：
1、节点自旋等待；
2、自选过程中如果发生了中断，则取消当前节点；
3、如果自旋达到阈值次数，则阻塞等待。

```
Object awaitFulfill(QNode s, E e, boolean timed, long nanos) {
    //计算等待截止时间：如果允许超时，则当前时间+超时时间；否则为0
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    //获得当前线程
    Thread w = Thread.currentThread();
    //获得需要自旋的次数
    //如果当前节点==head.next，表示等待时间最久的节点
    //如果允许超时，则自旋次数为maxTimedSpins，否则为maxUntimedSpins
    //如果当前节点!=head.next，则自旋值为0，即前面有等待的其他节点，不需要自旋
    int spins = ((head.next == s) ?
                 (timed ? maxTimedSpins : maxUntimedSpins) : 0);
    //自旋，死循环，直到发生中断退出或者配对的请求达到
    for (;;) {
        //如果当前线程发生了中断，则cas设置QNode的item为s（即当前QNode）
        if (w.isInterrupted())
            s.tryCancel(e);
        Object x = s.item;
        //QNode的item发生改变，发生了中断或者配对的请求已经到达
        if (x != e)
            return x;
        //允许超时情况下
        if (timed) {
            //是否超时
            nanos = deadline - System.nanoTime();
            //当前节点等待时间已经超时了
            if (nanos <= 0L) {
                //同中断时一样，设置item为当前QNode,重新自旋，下一循环会返回
                s.tryCancel(e);
                continue;
            }
        }
        //计算自旋次数（当前节点为head.next）
        if (spins > 0)
            --spins;
        //不允许自旋（非head.next，不需要自旋）或者自旋次数已经达到最大值，设置QNode的等待线程值为当前线程
        else if (s.waiter == null)
            s.waiter = w;
        //设置了等待线程，且没有设置超时时间，则阻塞当前线程（不设置超时时间的阻塞）
        else if (!timed)
            LockSupport.park(this);
        //如果剩余的超时等待时间大于自旋阈值时间，则阻塞当前线程（设置超时时间的阻塞）
        else if (nanos > spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanos);
    }
}

//将当前QNode的item值设置为QNode自己
void tryCancel(Object cmp) {
    UNSAFE.compareAndSwapObject(this, itemOffset, cmp, this);
}
```

#### TransferStack.transfer

```

```


配对节点类型：
- REQUEST，consumer
- DATA，producer
- FULFILLING
