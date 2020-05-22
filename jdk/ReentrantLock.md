## ReentrantLock源码分析

> 基于内部类Sync实现AQS的来实现锁，包括公平锁和非公平锁，默认是非公平锁，是可重入锁，在使用过程中lock和unlock成对出现。

### Sync

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = -5179523762034025860L;

    //加锁方法，由子类实现
    abstract void lock();

    //非阻塞形式非公平锁获取
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        //获取AQS的state值
        int c = getState();
        //state=0表示没有其他线程获取锁
        if (c == 0) {
            //cas设置锁的状态，设值成功则加锁成功
            if (compareAndSetState(0, acquires)) {
                //设置当前持有锁的线程（独占锁）
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        //表示当前有线程持有锁，判断持有锁得线程是否当前线程，如果是当前线程，可重入
        else if (current == getExclusiveOwnerThread()) {
            //改变state值
            int nextc = c + acquires;
            //溢出：重入次数过多，即加锁层次太多
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            //设置state
            setState(nextc);
            return true;
        }
        return false;
    }

    //释放锁
    protected final boolean tryRelease(int releases) {
        //可能完全释放或者重入的情况下释放锁
        int c = getState() - releases;
        //必须持有锁的线程才能释放锁
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        //锁完全释放
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }

    //判断是否当前线程持有锁
    protected final boolean isHeldExclusively() {
        return getExclusiveOwnerThread() == Thread.currentThread();
    }

    //创建等待条件
    final ConditionObject newCondition() {
        return new ConditionObject();
    }

    //返回持有锁的线程
    final Thread getOwner() {
        return getState() == 0 ? null : getExclusiveOwnerThread();
    }

    //返回当前线程加锁次数
    final int getHoldCount() {
        return isHeldExclusively() ? getState() : 0;
    }

    //判断是否已加锁
    final boolean isLocked() {
        return getState() != 0;
    }
}
```

#### NonfairSync
```java
static final class NonfairSync extends Sync {
    //加锁
    final void lock() {
        //非公平锁实现就是当前线程首先cas尝试加锁，与当前等待队列中的线程争夺锁
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            //执行AQS的获取资源方法
            acquire(1);
    }

    //尝试获取锁，非阻塞形式，成功则返回，否则获取失败
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
```

#### FairSync

```java
static final class FairSync extends Sync {

    //相比NonFairSync少了第一步cas操作，老老实实去争夺锁
    final void lock() {
        acquire(1);
    }

    //非阻塞形式获取公平锁：与nonfairTryAcquire唯一不同的是，当前线程获取锁时会先检查是否存在其他等待锁的线程，先到先得的原则
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            //首先检查是否存在正在等待锁的其他线程（依照公平锁的定义，按照先进先出，等待时间久的先获取锁），如果不存在其他等待锁的线程，则cas设置锁的状态加锁
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        //同
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}

//判断是否存在等待锁的前驱节点
public final boolean hasQueuedPredecessors() {
    Node t = tail;
    Node h = head;
    Node s;
    //存在等待所得前驱节点的条件：
    //1、h!=t表示当前队列已经初始化并且不为空，存在等待锁的线程
    //2、(s = h.next) == null，需要结合AQS的addWaiter方法和acquireQueued方法，具体分析如下：
    //如果当前只存在一个等待的线程，即head->curNode(tail)->null，而curNode正在已经获取锁，
    //此时执行setHead方法，会将head则会指向curNode,并将curNode的pre和thread置为null，即head的pre和thread为空，而此时没有其他线程入队，则head.next==curNode.next，即为null
    //3、s.thread != Thread.currentThread()表示等待的线程不是当前线程，存在其他节点
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

### ReentrantLock
> 以AQS的state值表示加锁状态和解锁状态：
0：表示无锁状态。
>=1：锁已被占有。

```java
public class ReentrantLock implements Lock, java.io.Serializable {
  private final Sync sync;//锁操作对象

  //默认非公平锁
  public ReentrantLock() {
      sync = new NonfairSync();
  }

  //设置公平锁、非公平锁
  public ReentrantLock(boolean fair) {
      sync = fair ? new FairSync() : new NonfairSync();
  }

  //加锁
  public void lock() {
      sync.lock();
  }

  //可接受中断的加锁
  public void lockInterruptibly() throws InterruptedException {
      sync.acquireInterruptibly(1);
  }

  //非阻塞获取锁
  public boolean tryLock() {
      return sync.nonfairTryAcquire(1);
  }

  //释放锁，state-1操作
  public void unlock() {
      sync.release(1);
  }

  //可设置超时时间的加锁
  public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
      return sync.tryAcquireNanos(1, unit.toNanos(timeout));
  }

  //创建条件队列
  public Condition newCondition() {
      return sync.newCondition();
  }
}
```
### ConditionObject
> AQS定义的条件队列。

```
public class ConditionObject implements Condition, java.io.Serializable {
    //条件队列头结点
    private transient Node firstWaiter;
    //条件队列尾结点
    private transient Node lastWaiter;

    public ConditionObject() { }

    //
    private Node addConditionWaiter() {
        Node t = lastWaiter;
        //如果最后的节点不为等待条件的节点，则移除等待队列中所有waitStatus!=CONDITION的节点
        if (t != null && t.waitStatus != Node.CONDITION) {
            //移除等待队列中不满足条件的节点
            unlinkCancelledWaiters();
            //移除后重新赋值
            t = lastWaiter;
        }
        //创建等待条件节点
        Node node = new Node(Thread.currentThread(), Node.CONDITION);
        //如果当前队列为空，赋值头节点，否则插入队列最后
        if (t == null)
            firstWaiter = node;
        else
            t.nextWaiter = node;
        lastWaiter = node;
        return node;
    }

    //唤醒等待队列中的节点
    //transferForSignal方法的作用是：设置当前节点的waitStatus=0，然后放入竞争资源的队列，即条件满足可以去重新获取锁了，并通知前驱节点后面有线程等待。
    //这时候，如果前驱节点的waitStatus>0，即已经取消，或者前驱节点的waitStatus<=0，但是cas设值前驱节点的waitStatus=SIGNAL时，对方的状态却变了（有可能已经获取锁），
    //此时唤醒当前线程去争夺锁
    private void doSignal(Node first) {
        do {
            if ( (firstWaiter = first.nextWaiter) == null)
                lastWaiter = null;
            first.nextWaiter = null;
        } while (!transferForSignal(first) &&
                 (first = firstWaiter) != null);
    }

    //清空条件队列，将所有条件队列中等待的线程节点放入等待锁的队列中去
    private void doSignalAll(Node first) {
        lastWaiter = firstWaiter = null;
        do {
            Node next = first.nextWaiter;
            first.nextWaiter = null;
            transferForSignal(first);
            first = next;
        } while (first != null);
    }

    //移除条件队列中waitStatus!=CONDITION的节点
    private void unlinkCancelledWaiters() {
        Node t = firstWaiter;
        Node trail = null;
        while (t != null) {
            Node next = t.nextWaiter;
            if (t.waitStatus != Node.CONDITION) {
                t.nextWaiter = null;
                if (trail == null)
                    firstWaiter = next;
                else
                    trail.nextWaiter = next;
                if (next == null)
                    lastWaiter = trail;
            }
            else
                trail = t;
            t = next;
        }
    }

    //唤醒条件队列中等待时间最长的线程（将头结点线程放入加锁队列中去）
    //唤醒操作需要在持有锁的情况下才能操作：可以理解为只有持有锁（当前占有资源），才有资格去唤醒其他线程
    public final void signal() {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        Node first = firstWaiter;
        if (first != null)
            doSignal(first);
    }

    //同signal类似，只是唤醒所有登台条件队列的线程（将条件队列中的所有线程放入加锁队列）
    public final void signalAll() {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        Node first = firstWaiter;
        if (first != null)
            doSignalAll(first);
    }

    //将当前线程放入到条件队列中去，释放持有的所有锁
    public final void awaitUninterruptibly() {
        Node node = addConditionWaiter();
        int savedState = fullyRelease(node);
        boolean interrupted = false;
        //判断当前节点是在CLS队列还是条件队列（因为多线程环境下，当前节点放入了条件队列，可能执行到此步时已经被其他线程唤醒放入了CLS队列），
        //如果还是在等待队列，则阻塞当前线程
        while (!isOnSyncQueue(node)) {
            LockSupport.park(this);
            if (Thread.interrupted())
                interrupted = true;
        }
        //当前线程被唤醒后尝试获取锁，可产生中断
        if (acquireQueued(node, savedState) || interrupted)
            selfInterrupt();
    }

    //重新中断
    private static final int REINTERRUPT =  1;
    //抛出中断
    private static final int THROW_IE    = -1;

    //判断线程在condition队列被阻塞的过程中，有没有被其他线程触发过中断请求
    //判断当前线程是否中断，如果没有发生中断，则返回0，否则：
    //1、如果将当前节点放入了CLS队列，则返回THROW_IE；
    //2、否则返回让出CPU执行权，返回REINTERRUPT
    private int checkInterruptWhileWaiting(Node node) {
        return Thread.interrupted() ?
            (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
            0;
    }

    //判断是抛出中断异常还是重新中断
    private void reportInterruptAfterWait(int interruptMode)
        throws InterruptedException {
        if (interruptMode == THROW_IE)
            throw new InterruptedException();
        else if (interruptMode == REINTERRUPT)
            selfInterrupt();
    }

    //等待的操作
    public final void await() throws InterruptedException {
        //如果发生中断，抛出中断异常
        if (Thread.interrupted())
            throw new InterruptedException();
        //加入条件队列
        Node node = addConditionWaiter();
        //释放持有的锁
        int savedState = fullyRelease(node);
        int interruptMode = 0;
        //判断是在CLS队列还是条件队列，isOnSyncQueue=true表示CLS队列，false表示条件队列
        //第一次判断肯定是false，因为前面已经释放锁了
        //如果后面被唤醒了（有线程执行了signal或者signalAll），已经加入到了CLS队列，就不会再执行循环和阻塞
        while (!isOnSyncQueue(node)) {
            //如果调用park()之前调用了unpark或者interrupt则park直接返回，不会挂起
            //阻塞当前线程
            //第一次总是park自己，开始阻塞等待
            LockSupport.park(this);
            //判断线程是正常唤醒还是被中断，如果是被中断了则放入CLS队列重新竞争锁；
            //如果是正常唤醒的，在其他线程执行doSignal方法时，会将当前节点加入CLS队列
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
        }

        //线程被唤醒后，会尝试获取锁，如果检查到发生了中断，设置中断模式
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        //transferAfterCancelledWait在中断的时候会将当前节点放入CLS队列，所以nextWaiter属性可能不为null
        //不为null，就需要移除waitStatus=CANCELLED的节点
        if (node.nextWaiter != null)
            unlinkCancelledWaiters();
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
    }

    //设置超时时间（多长时间后超时）的阻塞等待，除了判断是否超时，其他与wait()方法一样
    public final long awaitNanos(long nanosTimeout)
            throws InterruptedException {
        ...
    }

    //设置超时截止时间（到哪个时间点截止）的阻塞等待
    public final boolean awaitUntil(Date deadline)
            throws InterruptedException {
        ...
    }

    //判断当前条件队列是否当前对象持有
    final boolean isOwnedBy(AbstractQueuedSynchronizer sync) {
        return sync == AbstractQueuedSynchronizer.this;
    }

    //判断条件队列是否存在等待的线程
    protected final boolean hasWaiters() {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
            if (w.waitStatus == Node.CONDITION)
                return true;
        }
        return false;
    }
  }
```
### AQS之isOnSyncQueue
```
//判断当前节点是否在CLS队列，即：是在CLS队列还是在条件队列
final boolean isOnSyncQueue(Node node) {
    //如果节点的waitStatus==CONDITION，肯定在条件队列
    //如果节点的waitStatus!=CONDITION，而且pre==null肯定也在条件队列，因为条件队列为单向列表，节点的pre肯定为null，
    //至于CLS队列head的pre也为null，但是head表示当前已经获得锁的节点，此处显然不可能
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    //前面已经判断waitStatus!=CONDITION，而且节点的pre!=null，那么节点肯定在CLS队列中
    //当然也存在这种情况：当前节点已经释放锁了，成为了head节点，此时head的下一节点已经获得锁，并且head指向了下一节点，此时next也为null，就需要执行findNodeFromTail
    if (node.next != null)
        return true;
    //从CLS队列尾结点开始查找
    return findNodeFromTail(node);
}
```

### AQS之transferAfterCancelledWait

```
//正常情况下是当前节点被其他线程唤醒后，将当前节点从条件队列放入CLS队列
final boolean transferAfterCancelledWait(Node node) {
    //设置waitStatus=0，成功后将其放入CLS队列
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        enq(node);
        return true;
    }

    //其实如果调用signal唤醒，则线程已经在CLS队列，该步循环判断是否在CLS队列上，如果不在让出CPU执行权
    while (!isOnSyncQueue(node))
        Thread.yield();
    return false;
}
```
