## AbstractQueuedSynchronizer源码分析

> AQS是java.util.concurrent并发包的中实现锁ReetrantLock及同步器CountdownLatch、CyclicBarrier、Semaphore等的基础框架，基于volatile变量state和队列实现加锁、释放锁和阻塞等待的操作。

先挂一张获取资源的流程图，如下：


### Node定义
> 队列节点类，等待线程节点构成双向队列（包括独占节点和共享节点），结构如下：
      +------+  prev +-----+       +-----+
 head |      | <---- |     | <---- |     |  tail
      +------+       +-----+       +-----+

```java
static final class Node {
  //标识节点为共享锁
  static final Node SHARED = new Node();
  //标识节点为独占锁
  static final Node EXCLUSIVE = null;

  //waitStatus状态，标识当前节点因为超时或者中断，需要从同步队列中移除
  static final int CANCELLED =  1;
  //waitStatus状态，标识当前节点在释放锁或者执行取消动作时，需要唤醒继任节点，后继节点入队时，会将前继节点设置为SIGNAL
  static final int SIGNAL    = -1;
  //waitStatus状态，标识节点正等待在条件队列中，当其他线程调度了CONDITION的signal方法后，CONDITION状态的节点会从等待队列转移到同步队列中
  static final int CONDITION = -2;
  //waitStatus状态，标识共享模式下，释放锁操作应该传播到其他节点，该状态值是在doReleaseShared中设置的。
  static final int PROPAGATE = -3;
  //标识节点状态，初始为0，包括CANCELLED，SIGNAL，CONDITION，PROPAGATE，0这几种状态
  volatile int waitStatus;
  //前驱节点
  volatile Node prev;
  //继任节点
  volatile Node next;
  //持有当前节点的线程
  volatile Thread thread;
  //等待队列中的下一个节点，或者特殊的共享节点
  Node nextWaiter;

  //判断当前节点是否共享节点
  final boolean isShared() {
    return nextWaiter == SHARED;
  }

  //返回前驱节点
  final Node predecessor() throws NullPointerException {
      Node p = prev;
      if (p == null)
          throw new NullPointerException();
      else
          return p;
  }
}
```

### AQS定义
> AQS将state和head、tail使用volatile修饰，保证多线程下内存可见性。主要包括：
1、独占锁方法acquire、tryAcquire、release及tryRelease；
2、共享锁方法acquireShared、tryAcquireShared、releaseShared及tryReleaseShared；
3、带抛出中断异常的独占锁和共享锁方法；
4、带有超时的独占锁和共享锁方法；
其中try开头方法由继承类实现。

AQS定义的主要属性如下：
```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {

  //头结点
  private transient volatile Node head;
  //尾结点
  private transient volatile Node tail;
  //同步器状态标识
  private volatile int state;

  //自旋等待超时的阈值
  static final long spinForTimeoutThreshold = 1000L;
}
```

## 独占资源

### acquire
> 获取资源state的操作，其实就是给state设值，各同步器实现都是围绕state取值来判断的，其中tryAcquire由继承类实现

```java
  //获取资源，参数可以表示为需要获取的资源数
  public final void acquire(int arg) {
      //为非公平模式，线程过来先尝试获取资源，如果失败，则添加到等待队列中，并且自旋尝试获取资源（通过自旋产生阻塞，要么获取成功结束，要么产生中断结束）
      if (!tryAcquire(arg) &&
          acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
          //acquireQueued返回结果表示是否中断，到此步表示需要中断，执行中断操作
          selfInterrupt();
  }
```

#### addWaiter
> 创建一个指定模式（独占或者共享）的节点，并将它添加到等待队列中。

```java
  //当前线程获取资源失败，则将其作为一个新节点添加到队列中去阻塞等待
  private Node addWaiter(Node mode) {
      Node node = new Node(Thread.currentThread(), mode);
      //先尝试快速入队：即直接添加到尾结点
      Node pred = tail;
      //如果尾结点不为空，表示当前队列已经初始化，直接将当前节点添加到尾结点，否则执行enq操作
      if (pred != null) {
          node.prev = pred;
          //将当前节点设置为尾结点，如果操作失败，表示存在竞争，尾结点已经变化，执行enq操作
          if (compareAndSetTail(pred, node)) {
              pred.next = node;
              return node;
          }
      }
      //入队
      enq(node);
      return node;
  }

  //将当前节点添加到等待队列
  //1、如果当前队列为空，则初始化队列：头结点和尾结点都指向空节点，然后再执行2；
  //2、如果尾结点不为空，则执行入队操作
  private Node enq(final Node node) {
      //自旋直到成功添加到等待队列
      for (;;) {
          Node t = tail;
          //尾结点为空，表示队列尚未初始化，执行初始化操作
          if (t == null) {
              //将空节点设置为头结点
              if (compareAndSetHead(new Node()))
                  tail = head;
          } else {
              node.prev = t;
              if (compareAndSetTail(t, node)) {
                  t.next = node;
                  return t;
              }
          }
      }
  }
```
### acquireQueued
> 线程自旋尝试获取资源返回，或者设值前驱节点的waitStatus=SIGNAL，然后阻塞当前线程，等待前驱节点释放资源是唤醒或者中断，具体执行过程如下：
1、当前节点的前驱节点为头结点，则获取资源，如果获取成功，将当前节点设为头结点，流程结束并返回中断标识，否则执行第2步（shouldParkAfterFailedAcquire）；
2、判断前驱节点的waitStatus，如果waitStatus=SIGNAL，返回值为true，执行第5步，否则视waitStatus取值执行第3步或第4步；
3、如果前驱节点的waitStatus>0，标识该节点已经取消获取资源，则从前驱节点开始往前移除waitStatus=CANCELLED的节点，最后返回值为false，返回执行第1步（注：接下来的操作就是要么第一步成功获取资源返回，要么获取资源失败，执行第2步或第4步）；
4、如果waitStatus=0或waitStatus=PROPAGATE，设置前驱节点的waitStatus=SIGNAL，最后返回值为false，返回执行第1步（注：接下来要么第1步过程中成功获取资源返回，要么执行第2步，因为已经设置前驱节点的waitStatus=SIGNAL，然后执行第5步）；
5、当前节点没有获取资源，并且前驱节点的waitStatus已经设置为SIGNAL，则阻塞当前线程，返回值为线程的中断状态，即如果发生中断，则为true，否则为false；
总结：acquireQueued的执行流程就是要么获取资源返回，要么设置前驱节点waitStatus为SIGNAL，执行阻塞操作，最终返回值为当前线程的中断标识（true-发生了中断；false-未发生中断）。

```java
  final boolean acquireQueued(final Node node, int arg) {
      boolean failed = true;
      try {
          //中断标识
          boolean interrupted = false;
          //自旋
          for (;;) {
              //获得当前节点的前驱节点
              final Node p = node.predecessor();
              //判断前驱节点是否为头结点，如果前驱节点为头结点，表示当前节点是可以获取资源的线程，尝试获取资源
              if (p == head && tryAcquire(arg)) {
                  //获取资源成功，将head指向当前节点（head节点为空节点，会设置节点的thread和prev为空）
                  setHead(node);
                  //因为此处p为头结点，当前节点获取资源成功后，将head指向了当前节点，将原先的头结点next设为空，即从当前队列中移除（其prev也为空，节点没了引用和被引用），帮助gc
                  p.next = null;
                  failed = false;
                  //返回中断标识
                  return interrupted;
              }
              //当前节点的前驱节点不为头结点，则需要根据前驱节点的waitStatus取值判断执行的操作，分为以下几种情况：
              //1、前驱节点的waitStatus=SIGNAL，表示此节点已经设置成唤醒后继节点的状态，shouldParkAfterFailedAcquire返回值为true，当前节点作为后继节点可以阻塞等待前驱节点的唤醒或者发生中断，即执行parkAndCheckInterrupt；
              //2、前驱节点的waitStatus<0，向前移除包括前驱节点在内所有waitStatus=CANCELLED的节点，shouldParkAfterFailedAcquire返回false，继续自旋；
              //3、前驱节点的waitStatus=0或waitStatus=PROPAGATE，设置前驱节点的waitStatus为SIGNAL，shouldParkAfterFailedAcquire返回false，继续自旋；
              if (shouldParkAfterFailedAcquire(p, node) &&
                  parkAndCheckInterrupt())
                  //此处表示parkAndCheckInterrupt检测到当前线程发生了中断
                  interrupted = true;
          }
      } finally {
          //获取失败：finally在return前始终执行，正常执行时failed会设置为false，如果tryAcquire发生异常failed才为true，取消当前节点获取资源的操作
          if (failed)
              cancelAcquire(node);
      }
  }

  //将节点设为头结点（空节点，只有next指向）
  private void setHead(Node node) {
      head = node;
      node.thread = null;
      node.prev = null;
  }

  //找到waitStatus=SIGNAL的前驱节点或者设值前驱节点为SIGNAL
  private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        //如果正好前驱节点的waitStatus=SIGNAL，标识前驱节点在释放资源时会唤醒当前节点，则直接返回true，去执行阻塞
        if (ws == Node.SIGNAL)
            return true;
        //标识前驱节点已经被取消了
        if (ws > 0) {
            //往前移除所有waitStatus=CANCELLED的节点
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            //设值前驱节点的waitStatus=SIGNAL
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        //返回false会继续自旋，再次尝试获取资源
        return false;
    }

  //阻塞当前线程，唤醒后返回中断标识
  private final boolean parkAndCheckInterrupt() {
      LockSupport.park(this);
      return Thread.interrupted();
  }
```

### release
> 释放资源，会调用tryRelease，由继承类实现具体的释放资源操作。

```java
  public final boolean release(int arg) {
      //释放资源
      if (tryRelease(arg)) {
          //h != null && h.waitStatus != 0表示head节点已经设置为获取资源的线程节点：因为当前节点获取资源后，会将head指向当前节点，waitStatus=0是初始化状态
          Node h = head;
          if (h != null && h.waitStatus != 0)
              //唤醒后继节点，然后后继节点会接着执行acquireQueued后续操作：尝试获取资源（因为后继节点已经是head的后继节点，公平锁下会获取资源）或者中断
              unparkSuccessor(h);
          return true;
      }
      return false;
  }

  //唤醒第一个正常后继节点
  private void unparkSuccessor(Node node) {

      int ws = node.waitStatus;
      //表示waitStatus=SIGNAL或PROPAGATE，则将节点的waitStatus置为初始化状态，其实就是变为空节点（head初始为空节点）
      if (ws < 0)
          compareAndSetWaitStatus(node, ws, 0);

      Node s = node.next;
      //后继节点为空或者已经取消
      if (s == null || s.waitStatus > 0) {
          s = null;
          //从队列尾部往前查找：找到后继节点中第一个没被取消的节点
          for (Node t = tail; t != null && t != node; t = t.prev)
              if (t.waitStatus <= 0)
                  s = t;
      }
      //如果存在正常的后继节点，执行唤醒操作
      if (s != null)
          LockSupport.unpark(s.thread);
  }
```

### acquireInterruptibly
> 带有中断异常的获取资源方法：获取资源过程中，如果发生中断，则抛出中断异常。

```java
  public final void acquireInterruptibly(int arg)
            throws InterruptedException {
      //判断线程的中断标识，如果中断直接抛出中断异常
      if (Thread.interrupted())
          throw new InterruptedException();
      //获取资源
      if (!tryAcquire(arg))
          //执行可中断的获取资源
          doAcquireInterruptibly(arg);
  }

  //自旋获取资源，操作过程与acquireQueued类似，唯一区别就是如果中间发生中断，就会抛出中断异常
  private void doAcquireInterruptibly(int arg)
        throws InterruptedException {
      final Node node = addWaiter(Node.EXCLUSIVE);
      boolean failed = true;
      try {
          for (;;) {
              final Node p = node.predecessor();
              if (p == head && tryAcquire(arg)) {
                  setHead(node);
                  p.next = null; // help GC
                  failed = false;
                  return;
              }
              if (shouldParkAfterFailedAcquire(p, node) &&
                  parkAndCheckInterrupt())
                  throw new InterruptedException();
          }
      } finally {
          if (failed)
              cancelAcquire(node);
      }
  }
```

### cancelAcquire
> 取消获取资源的操作，外部无法调用。

```java
  private void cancelAcquire(Node node) {
      if (node == null)
          return;

      node.thread = null;

      // 跳过所有已经取消的前驱节点，协助删除取消节点的操作
      Node pred = node.prev;
      while (pred.waitStatus > 0)
          node.prev = pred = pred.prev;

      //记录找到的前驱节点的next（用于cas时判断expect值），如果前驱节点的waitStatus=CANCELLED，pred.next并不为当前节点
      Node predNext = pred.next;

      //设置当前节点为取消状态
      node.waitStatus = Node.CANCELLED;

      //如果当前节点为尾结点，直接设置找到的pred节点为尾结点
      if (node == tail && compareAndSetTail(node, pred)) {
          //设置尾结点成功，则next设为null
          compareAndSetNext(pred, predNext, null);
      } else {
          int ws;
          //如果找到的前驱节点不为头结点（即不是当前获取资源的节点），并且该节点的waitStatus为SIGNAL或者将其设置为SIGNAL，则设置当前节点的后继节点为前驱节点的next
          if (pred != head &&
              ((ws = pred.waitStatus) == Node.SIGNAL ||
               (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
              pred.thread != null) {
              Node next = node.next;
              //后继节点正常，等待唤醒
              if (next != null && next.waitStatus <= 0)
                  compareAndSetNext(pred, predNext, next);
          } else {//前驱节点为头结点，唤醒后继节点
              unparkSuccessor(node);
          }
          //当前节点指向自己，即取消应用，协助gc
          node.next = node;
      }
  }
```

## 共享资源
### acquireShared
> 与独占锁操作的区别是获取资源后的操作，多了setHeadAndPropagate操作。

```java
  public final void acquireShared(int arg) {
      //子类实现过程中，返回值小于0表示获取共享资源失败
      if (tryAcquireShared(arg) < 0)
          doAcquireShared(arg);
  }

  //获取共享资源
  private void doAcquireShared(int arg) {
      //新建共享节点，并加入到共享节点队列
      final Node node = addWaiter(Node.SHARED);
      boolean failed = true;
      try {
          boolean interrupted = false;
          for (;;) {
              final Node p = node.predecessor();
              //当前节点的前驱节点为head
              if (p == head) {
                  //获取资源
                  int r = tryAcquireShared(arg);
                  //获取资源成功
                  if (r >= 0) {
                      setHeadAndPropagate(node, r);
                      p.next = null; // help GC
                      if (interrupted)
                          selfInterrupt();
                      failed = false;
                      return;
                  }
              }
              //与acquireQueued操作一致
              if (shouldParkAfterFailedAcquire(p, node) &&
                  parkAndCheckInterrupt())
                  interrupted = true;
          }
      } finally {
          if (failed)
              cancelAcquire(node);
      }
  }

  private void setHeadAndPropagate(Node node, int propagate) {
      //标识旧的head
      Node h = head;
      //将head引用指向当前节点
      setHead(node);

      //无论当前节点获取资源成功或者失败，如果失败则判断旧的头结点和置为新头结点的当前节点的waitStatus
      //1、当前线程获取了共享资源，执行3
      //2、当前线程没有获取共享资源，头结点为空或头节点waitStatus=SIGNAL或waitStatus=PROPAGATE，执行3
      //3、唤醒后继共享节点或者设值头结点的waitStatus=PROPAGATE
      if (propagate > 0 || h == null || h.waitStatus < 0 ||
          (h = head) == null || h.waitStatus < 0) {
          Node s = node.next;
          if (s == null || s.isShared())
              //设置节点的传播状态，执行到此步操作时，头节点为当前节点，且当前节点的waitStatus=SIGNAL，在doReleaseShared中会唤醒后继节点
              doReleaseShared();
      }
  }
```
### releaseShared
> 释放共享资源，具体的由子类实现。

```java
  public final boolean releaseShared(int arg) {
      if (tryReleaseShared(arg)) {
          doReleaseShared();
          return true;
      }
      return false;
  }

  private void doReleaseShared() {
      for (;;) {
          Node h = head;
          if (h != null && h != tail) {
              int ws = h.waitStatus;
              //如果头结点waitStatus=SIGNAL，则waitStatus设为0，即空节点，且唤醒后继节点
              //节点取消cancelAcquire操作时会设置前驱的waitStatus=SIGNAL
              if (ws == Node.SIGNAL) {
                  if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                      continue;            // loop to recheck cases
                  unparkSuccessor(h);
              }//初始状态下，waitStatus=0,设值waitStatus=PROPAGATE，最终PROPAGATE会转化为SIGNAL（由获取资源的acquire方法的shouldParkAfterFailedAcquire调用保证）
              else if (ws == 0 &&
                       !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                  continue;                // loop on failed CAS
          }
          if (h == head)                   // loop if head changed
              break;
      }
  }
```

> AQS为并发包的基础，acquire操作包含的细节较多，资源获取，唤醒后继节点，协助移除CANCELLED节点，构思设计精巧，为Doug Lea大神点赞。
