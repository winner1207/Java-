# ReentrantLock
java中有两类锁，一类是Synchronized，而另一类就是J.U.C中提供的锁。ReentrantLock与Synchronized都是可重入锁，本质上都是lock与unlock的操作。接下来我们介绍三种J.U.C中的锁，其中 ReentrantLock使用synchronized与之比对介绍。

# ReentrantLock与synchronized的区别
- 可重入性：两者的锁都是可重入的，差别不大，有线程进入锁，计数器自增1，等下降为0时才可以释放锁
- 锁的实现：synchronized是基于JVM实现的（用户很难见到，无法了解其实现），ReentrantLock是JDK实现的。
- 性能区别：在最初的时候，二者的性能差别差很多，当synchronized引入了偏向锁、轻量级锁（自选锁）后，二者的性能差别不大，官方推荐synchronized（写法更容易、在优化时其实是借用了ReentrantLock的CAS技术，试图在用户态就把问题解决，避免进入内核态造成线程阻塞）
- 功能区别： 
（1）便利性：synchronized更便利，它是由编译器保证加锁与释放。ReentrantLock是需要手动释放锁，所以为了避免忘记手工释放锁造成死锁，所以最好在finally中声明释放锁。 
（2）锁的细粒度和灵活度，ReentrantLock优于synchronized

# ReentrantLock独有的功能
- 可以指定是公平锁还是非公平锁，sync只能是非公平锁。（所谓公平锁就是先等待的线程先获得锁）
- 提供了一个Condition类，可以分组唤醒需要唤醒的线程。不像是synchronized要么随机唤醒一个线程，要么全部唤醒。
- 提供能够中断等待锁的线程的机制，通过lock.lockInterruptibly()实现，这种机制 ReentrantLock是一种自选锁，通过循环调用CAS操作来实现加锁。性能比较好的原因是避免了进入内核态的阻塞状态。

# 要放弃synchronized？
从上边的介绍，看上去ReentrantLock不仅拥有synchronized的所有功能，而且有一些功能synchronized无法实现的特性。性能方面，ReentrantLock也不比synchronized差，那么到底我们要不要放弃使用synchronized呢？答案是不要这样做。

J.U.C包中的锁定类是用于高级情况和高级用户的工具，除非说你对Lock的高级特性有特别清楚的了解以及有明确的需要，或这有明确的证据表明同步已经成为可伸缩性的瓶颈的时候，否则我们还是继续使用synchronized。相比较这些高级的锁定类，synchronized还是有一些优势的，比如synchronized不可能忘记释放锁。还有当JVM使用synchronized管理锁定请求和释放时，JVM在生成线程转储时能够包括锁定信息，这些信息对调试非常有价值，它们可以标识死锁以及其他异常行为的来源。

# 如何使用ReentrantLock？
<pre>
//创建锁：使用Lock对象声明，使用ReentrantLock接口创建
private final static Lock lock = new ReentrantLock();
//使用锁：在需要被加锁的方法中使用
private static void add() {
    lock.lock();
    try {
        count++;
    } finally {
        lock.unlock();
    }
}
</pre>

分析一下源码：

<pre>
//初始化方面：
//在new ReentrantLock的时候默认给了一个不公平锁
public ReentrantLock() {
    sync = new NonfairSync();
}
//也可以加参数来初始化指定使用公平锁还是不公平锁
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}	
</pre>

## 内置函数（部分）

基础特性：

- tryLock()：仅在调用时锁定未被另一个线程保持的情况下才获取锁定。
- tryLock(long timeout, TimeUnit unit)：如果锁定在给定的时间内没有被另一个线程保持且当前线程没有被中断，则获取这个锁定。
- lockInterruptbily：如果当前线程没有被中断的话，那么就获取锁定。如果中断了就抛出异常。
- isLocked：查询此锁定是否由任意线程保持
- isHeldByCurrentThread：查询当前线程是否保持锁定状态。
- isFair：判断是不是公平锁 
…


Condition相关特性：

- hasQueuedThread(Thread)：查询指定线程是否在等待获取此锁定
- hasQueuedThreads()：查询是否有线程在等待获取此锁定
- getHoldCount()：查询当前线程保持锁定的个数，也就是调用Lock方法的个数 
…

## Condition的使用
Condition可以非常灵活的操作线程的唤醒，下面是一个线程等待与唤醒的例子，其中用1234序号标出了日志输出顺序

<pre>
public static void main(String[] args) {
    ReentrantLock reentrantLock = new ReentrantLock();
    Condition condition = reentrantLock.newCondition();//创建condition
    //线程1
    new Thread(() -> {
        try {
            reentrantLock.lock();
            log.info("wait signal"); // 1
            condition.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.info("get signal"); // 4
        reentrantLock.unlock();
    }).start();
    //线程2
    new Thread(() -> {
        reentrantLock.lock();
        log.info("get lock"); // 2
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        condition.signalAll();//发送信号
        log.info("send signal"); // 3
        reentrantLock.unlock();
    }).start();
}
</pre>

（这里对等待队列不熟悉的，请回顾我的上一篇文章中讲解的AQS等待队列：高并发探索（十一）：并发容器J.U.C – AQS组件CountDownLatch、Semaphore、CyclicBarrier） 
输出过程讲解：

1. 线程1调用了reentrantLock.lock()，线程进入AQS等待队列，输出1号log 
2. 接着调用了awiat方法，线程从AQS队列中移除，锁释放，直接加入condition的等待队列中 
3. 线程2因为线程1释放了锁，拿到了锁，输出2号log 
4. 线程2执行condition.signalAll()发送信号，输出3号log 
5. condition队列中线程1的节点接收到信号，从condition队列中拿出来放入到了AQS的等待队列,这时线程1并没有被唤醒。 
6. 线程2调用unlock释放锁，因为AQS队列中只有线程1，因此AQS释放锁按照从头到尾的顺序，唤醒线程1 
7. 线程1继续执行，输出4号log，并进行unlock操作。

# 读写锁：ReentrantReadWriteLock读写锁
在没有任何读写锁的时候才可以取得写入锁(悲观读取，容易写线程饥饿)，也就是说如果一直存在读操作，那么写锁一直在等待没有读的情况出现，这样我的写锁就永远也获取不到，就会造成等待获取写锁的线程饥饿。 
平时使用的场景并不多。

<pre>
public class LockExample3 {

    private final Map<String, Data> map = new TreeMap<>();
    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
    private final Lock readLock = lock.readLock();//读锁
    private final Lock writeLock = lock.writeLock();//写锁

    //加读锁
    public Data get(String key) {
        readLock.lock();
        try {
            return map.get(key);
        } finally {
            readLock.unlock();
        }
    }
    //加写锁
    public Data put(String key, Data value) {
        writeLock.lock();
        try {
            return map.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }

    class Data {}
}
</pre>

# 票据锁：StempedLock
它控制锁有三种模式（写、读、乐观读）。一个StempedLock的状态是由版本和模式两个部分组成。锁获取方法返回一个数字作为票据（stamp），他用相应的锁状态表示并控制相关的访问。数字0表示没有写锁被锁写访问，在读锁上分为悲观锁和乐观锁。

> 乐观读： 如果读的操作很多写的很少，我们可以乐观的认为读的操作与写的操作同时发生的情况很少，因此不悲观的使用完全的读取锁定。程序可以查看读取资料之后是否遭到写入资料的变更，再采取之后的措施。

如何使用？

<pre>
//定义
private final static StampedLock lock = new StampedLock();
//需要上锁的方法
private static void add() {
    long stamp = lock.writeLock();
    try {
        count++;
    } finally {
        lock.unlock(stamp);
    }
}
</pre>

分析一下源码：

<pre>
class Point {
        private double x, y;
        private final StampedLock sl = new StampedLock();

        void move(double deltaX, double deltaY) {
            long stamp = sl.writeLock();
            try {
                x += deltaX;
                y += deltaY;
            } finally {
                sl.unlockWrite(stamp);
            }
        }

        //下面看看乐观读锁案例
        double distanceFromOrigin() { // A read-only method
            long stamp = sl.tryOptimisticRead(); //获得一个乐观读锁
            double currentX = x, currentY = y;  //将两个字段读入本地局部变量
            if (!sl.validate(stamp)) { //检查发出乐观读锁后同时是否有其他写锁发生？
                stamp = sl.readLock();  //如果没有，我们再次获得一个读悲观锁
                try {
                    currentX = x; // 将两个字段读入本地局部变量
                    currentY = y; // 将两个字段读入本地局部变量
                } finally {
                    sl.unlockRead(stamp);
                }
            }
            return Math.sqrt(currentX * currentX + currentY * currentY);
        }

        //下面是悲观读锁案例
        void moveIfAtOrigin(double newX, double newY) { // upgrade
            // Could instead start with optimistic, not read mode
            long stamp = sl.readLock();
            try {
                while (x == 0.0 && y == 0.0) { //循环，检查当前状态是否符合
                    long ws = sl.tryConvertToWriteLock(stamp); //将读锁转为写锁
                    if (ws != 0L) { //这是确认转为写锁是否成功
                        stamp = ws; //如果成功 替换票据
                        x = newX; //进行状态改变
                        y = newY;  //进行状态改变
                        break;
                    } else { //如果不能成功转换为写锁
                        sl.unlockRead(stamp);  //我们显式释放读锁
                        stamp = sl.writeLock();  //显式直接进行写锁 然后再通过循环再试
                    }
                }
            } finally {
                sl.unlock(stamp); //释放读锁或写锁
            }
        }
    }
</pre>

# 如何选择锁？
- 1、当只有少量竞争者，使用synchronized 
- 2、竞争者不少但是线程增长的趋势是能预估的，使用ReetrantLock 
- 3、synchronized不会造成死锁，jvm会自动释放死锁。