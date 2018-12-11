# AQS简介

AQS全名：AbstractQueuedSynchronizer，是并发容器J.U.C（java.lang.concurrent）下locks包内的一个类。它实现了一个FIFO(FirstIn、FisrtOut先进先出)的队列。底层实现的数据结构是一个双向列表。 

![](https://img-blog.csdn.net/20180423180428455?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

- Sync queue：同步队列，是一个双向列表。包括head节点和tail节点。head节点主要用作后续的调度。 
- Condition queue：非必须，单向列表。当程序中存在condition的时候才会存在此列表。

# AQS设计思想

- 使用Node实现FIFO(first in、first out)队列，可以用于构建锁或者其他同步装置的基础框架。
- 利用int类型标识状态。在AQS类中有一个叫做state的成员变量

<pre>
/**
 * The synchronization state.
 */
private volatile int state;
</pre>

- 基于AQS有一个同步组件，叫做ReentrantLock。在这个组件里，state表示获取锁的线程数，假如state=0，表示还没有线程获取锁，1表示有线程获取了锁。大于1表示重入锁的数量。
- 继承：子类通过继承并通过实现它的方法管理其状态（acquire和release方法操纵状态）。
- 可以同时实现排它锁和共享锁模式（独占、共享），站在一个使用者的角度，AQS的功能主要分为两类：独占和共享。它的所有子类中，要么实现并使用了它的独占功能的api，要么使用了共享锁的功能，而不会同时使用两套api，即便是最有名的子类ReentrantReadWriteLock也是通过两个内部类读锁和写锁分别实现了两套api来实现的。

# AQS的大致实现思路

AQS内部维护了一个CLH队列来管理锁。线程会首先尝试获取锁，如果失败就将当前线程及等待状态等信息包装成一个node节点加入到同步队列sync queue里。 

接着会不断的循环尝试获取锁，条件是当前节点为head的直接后继才会尝试。如果失败就会阻塞自己直到自己被唤醒。而当持有锁的线程释放锁的时候，会唤醒队列中的后继线程。

# AQS组件：CountDownLatch

![](https://img-blog.csdn.net/20180423191843274?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

通过一个计数来保证线程是否需要被阻塞。实现一个或多个线程等待其他线程执行的场景。

我们定义一个CountDownLatch，通过给定的计数器为其初始化，该计数器是原子性操作，保证同时只有一个线程去操作该计数器。调用该类await方法的线程会一直处于阻塞状态。只有其他线程调用countDown方法（每次使计数器-1），使计数器归零才能继续执行。

示例代码：[CountDownLatchExample1.java](../src/main/java/com/mmall/concurrency/example/aqs/CountDownLatchExample1.java)

<pre>
final CountDownLatch countDownLatch = new CountDownLatch(threadCount);

for (int i = 0; i < threadCount; i++) {
    final int threadNum = i;
    exec.execute(() -> {
        try {
            test(threadNum);  //需要被等待的线程执行的方法
        } catch (Exception e) {
            log.error("exception", e);
        } finally {
            countDownLatch.countDown();
        }
    });
}
countDownLatch.await();
</pre>

CountDownLatch的await方法还有重载形式，可以设置等待的时间，如果超过此时间，计数器还未清零，则不继续等待：

示例代码：[CountDownLatchExample2.java](../src/main/java/com/mmall/concurrency/example/aqs/CountDownLatchExample2.java)

<pre>
countDownLatch.await(10, TimeUnit.MILLISECONDS);

//参数1：等待的时间长度
//参数2：等待的时间单位
</pre>

# AQS组件：Semaphore

- 用于保证同一时间并发访问线程的数目。
- 信号量在操作系统中是很重要的概念，Java并发库里的Semaphore就可以很轻松的完成类似操作系统信号量的控制。Semaphore可以很容易控制系统中某个资源被同时访问的线程个数。
- 在数据结构中我们学过链表，链表正常是可以保存无限个节点的，而Semaphore可以实现有限大小的列表。
- 使用场景：仅能提供有限访问的资源。比如数据库连接。
- Semaphore使用acquire方法和release方法来实现控制：

<pre>
/**
 * 1、普通调用
 */
try {
     semaphore.acquire(); // 获取一个许可
     test();//需要并发控制的内容
     semaphore.release(); // 释放一个许可
} catch (Exception e) {
     log.error("exception", e);
}

/**
 * 2、同时获取多个许可，同时释放多个许可
 */
 try {
     semaphore.acquire(2);
     test();
     semaphore.release(2);
} catch (Exception e) {
     log.error("exception", e);
}

/*
 * 3、尝试获取许可，获取不到不执行
 */
 try {
     if (semaphore.tryAcquire()) {
        test(threadNum);
        semaphore.release();
     }
 } catch (Exception e) {
     log.error("exception", e);
}

/*
 * 4、尝试获取许可一段时间，获取不到不执行
 * 参数1：等待时间长度  参数2：等待时间单位
 */
try {
     if (semaphore.tryAcquire(5000, TimeUnit.MILLISECONDS)) {
        test(threadNum);
        semaphore.release(); 
     }
} catch (Exception e) {
     log.error("exception", e);
}
</pre>

# AQS组件：CyclicBarrier

![](https://img-blog.csdn.net/20180423234242640?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

- 也是一个同步辅助类，它允许一组线程相互等待，直到到达某个公共的屏障点（循环屏障）
- 通过它可以完成多个线程之间相互等待，只有每个线程都准备就绪后才能继续往下执行后面的操作。
- 每当有一个线程执行了await方法，计数器就会执行+1操作，待计数器达到预定的值，所有的线程再同时继续执行。由于计数器释放之后可以重用（reset方法），所以称之为循环屏障。
- 与CountDownLatch区别： 
1、计数器可重复用 
2、描述一个或多个线程等待其他线程的关系/多个线程相互等待

<pre>
//公共线程循环调用方法
private static CyclicBarrier barrier = new CyclicBarrier(5);

public static void main(String[] args) throws Exception {
    ExecutorService executor = Executors.newCachedThreadPool();

    for (int i = 0; i < 10; i++) {
        final int threadNum = i;
        Thread.sleep(1000);
        executor.execute(() -> {
            try {
                race(threadNum);
            } catch (Exception e) {
                log.error("exception", e);
            }
        });
    }
    executor.shutdown();
}

//使用方法1：每个线程都持续等待
private static void race(int threadNum) throws Exception {
    Thread.sleep(1000);
    log.info("{} is ready", threadNum);
    barrier.await();
    log.info("{} continue", threadNum);
}

//使用方法2：每个线程只等待一段时间
private static void race(int threadNum) throws Exception {
    Thread.sleep(1000);
    try {
        barrier.await(2000, TimeUnit.MILLISECONDS);
    } catch (InterruptedException | BrokenBarrierException | TimeoutException e) {
        log.warn("BarrierException", e);
    }
}

//使用方法3：在初始化的时候设置runnable，当线程达到屏障时优先执行runnable
private static CyclicBarrier barrier = new CyclicBarrier(5, () -> {
    log.info("callback is running");
});
</pre>