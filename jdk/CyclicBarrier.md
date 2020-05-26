## CyclicBarrier源码分析

> 相比CountDownLatch基于AQS共享模式实现，CyclicBarrier使用了Condition实现，要简单很多，俗称“栅栏”，可以实现一组线程互相等待，当所有线程达到某个屏障点后再进行后续的动作。

### CyclicBarrier实现

> parties=n，表示有n个线程参与；
count初始值等于parties，随着每个线程执行await一次， count-1，穿过栅栏后重置为parties。

```
public class CyclicBarrier {

    //静态内部类
    private static class Generation {
        boolean broken = false;
    }

    //同步操作锁
    private final ReentrantLock lock = new ReentrantLock();
    //线程拦截器
    private final Condition trip = lock.newCondition();
    //每次拦截的线程数
    private final int parties;
    //换代前执行的任务
    private final Runnable barrierCommand;
    //表示栅栏的当前代
    private Generation generation = new Generation();

    //计数器
    private int count;

    private void nextGeneration() {
        trip.signalAll();
        count = parties;
        generation = new Generation();
    }

    private void breakBarrier() {
        generation.broken = true;
        count = parties;
        trip.signalAll();
    }

    //定义初始线程数
    public CyclicBarrier(int parties) {
        this(parties, null);
    }

    //定义初始线程数和换代前执行的任务
    //barrierAction表示在大家都到达栅栏，在通过之前需要执行的操作，由最后一个到达栅栏的线程执行。
    public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }

    //等待其他线程达到屏障点
    public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
    }

    //不管是正常await还是设置超时时间，执行的都是此方法，只是会根据标识timed判断处理
    //timed:是否设置超时时间
    //nanos：超时时间
    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        //执行await前首先获得锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            final Generation g = generation;
            //进行换代，无法继续执行
            if (g.broken)
                throw new BrokenBarrierException();

            //当前线程发生中断，break当前Generation，唤醒所有等待的线程
            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }

            //加锁条件下，此操作是原子的
            int index = --count;
            //达到换代条件
            if (index == 0) {
                boolean ranAction = false;
                try {
                    //如果存在换代前设置的任务，则执行
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run();
                    ranAction = true;
                    //换代初始化，唤醒所有等待的线程
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            //自旋直到所有线程达到屏障点，或者有线程发生中断，执行了breakBarrier，或者超时
            for (;;) {
                try {
                    //执行await，阻塞等待或者超时等待
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    //发生中断，执行换代初始化，抛出异常
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        //执行中断
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }

    //重置
    public void reset() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            breakBarrier();   // break the current generation
            nextGeneration(); // start a new generation
        } finally {
            lock.unlock();
        }
    }
}
```
