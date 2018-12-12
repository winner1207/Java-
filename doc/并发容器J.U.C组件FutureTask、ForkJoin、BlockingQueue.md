# FutureTask
FutureTask是J.U.C中的类，是一个可删除的异步计算类。这个类提供了Future接口的的基本实现，使用相关方法启动和取消计算，查询计算是否完成，并检索计算结果。只有在计算完成时才能使用get方法检索结果;如果计算尚未完成，get方法将会阻塞。一旦计算完成，计算就不能重新启动或取消(除非使用runAndReset方法调用计算)。

## Runnable与Callable对比
通常实现一个线程我们会使用继承Thread的方式或者实现Runnable接口，这两种方式有一个共同的缺陷就是在执行完任务之后无法获取执行结果。从Java1.5之后就提供了Callable与Future，这两个接口就可以实现获取任务执行结果。

- Runnable接口：代码非常简单，只有一个方法run

<pre>
public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}
</pre>

- Callable泛型接口：有泛型参数，提供了一个call方法，执行后可返回传入的泛型参数类型的结果

<pre>
public interface Callable<V> {
    V call() throws Exception;
}
</pre>

## Future接口
Future接口提供了一系列方法用于控制线程执行计算，如下：

<pre>
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);//取消任务
    boolean isCancelled();//是否被取消
    boolean isDone();//计算是否完成
    V get() throws InterruptedException, ExecutionException;//获取计算结果，在执行过程中任务被阻塞
    V get(long timeout, TimeUnit unit)//timeout等待时间、unit时间单位
        throws InterruptedException, ExecutionException, TimeoutException;
}
</pre>

使用方法：

<pre>
public class FutureExample {

    static class MyCallable implements Callable<String> {
        @Override
        public String call() throws Exception {
            log.info("do something in callable");
            Thread.sleep(5000);
            return "Done";
        }
    }

    public static void main(String[] args) throws Exception {
        ExecutorService executorService = Executors.newCachedThreadPool();
        Future<String> future = executorService.submit(new MyCallable());//线程池提交任务
        log.info("do something in main");
        Thread.sleep(1000);
        String result = future.get();//获取不到一直阻塞
        log.info("result：{}", result);
    }
}
</pre>

运行结果：阻塞效果 

![](https://img-blog.csdn.net/20180502170850681?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

## FutureTask
Future实现了RunnableFuture接口，而RunnableFuture接口继承了Runnable与Future接口，所以它既可以作为Runnable被线程中执行，又可以作为callable获得返回值。

<pre>
public class FutureTask<V> implements RunnableFuture<V> {
    ...
}

public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}
</pre>

FutureTask支持两种参数类型，Callable和Runnable，在使用Runnable 时，还可以多指定一个返回结果类型。


<pre>
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;       // ensure visibility of callable
}

public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}
</pre>

使用方法：

<pre>
public class FutureTaskExample {

    public static void main(String[] args) throws Exception {
        FutureTask<String> futureTask = new FutureTask<String>(new Callable<String>() {
            @Override
            public String call() throws Exception {
                log.info("do something in callable");
                Thread.sleep(5000);
                return "Done";
            }
        });

        new Thread(futureTask).start();
        log.info("do something in main");
        Thread.sleep(1000);
        String result = futureTask.get();
        log.info("result：{}", result);
    }
}
</pre>

运行结果： 

![](https://img-blog.csdn.net/20180502170946350?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

# ForkJoin
ForkJoin是Java7提供的一个并行执行任务的框架，是把大任务分割成若干个小任务，待小任务完成后将结果汇总成大任务结果的框架。主要采用的是工作窃取算法，工作窃取算法是指某个线程从其他队列里窃取任务来执行。 

![](https://img-blog.csdn.net/20180502173012969?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

在窃取过程中两个线程会访问同一个队列，为了减少窃取任务线程和被窃取任务线程之间的竞争，通常我们会使用双端队列来实现工作窃取算法。被窃取任务的线程永远从队列的头部拿取任务，窃取任务的线程从队列尾部拿取任务。

## 局限性：
1. 任务只能使用fork和join作为同步机制，如果使用了其他同步机制，当他们在同步操作时，工作线程就不能执行其他任务了。比如在fork框架使任务进入了睡眠，那么在睡眠期间内在执行这个任务的线程将不会执行其他任务了。 
2. 我们所拆分的任务不应该去执行IO操作，如读和写数据文件。 
3. 任务不能抛出检查异常。必须通过必要的代码来处理他们。

## 框架核心：
- 核心有两个类：ForkJoinPool | ForkJoinTask 
- ForkJoinPool：负责来做实现，包括工作窃取算法、管理工作线程和提供关于任务的状态以及他们的执行信息。 
- ForkJoinTask:提供在任务中执行fork和join的机制。

## 使用方式：（模拟加和运算）
<pre>
@Slf4j
public class ForkJoinTaskExample extends RecursiveTask<Integer> {

    public static final int threshold = 2;//设定不大于两个数相加就直接for循环，不适用框架
    private int start;
    private int end;

    public ForkJoinTaskExample(int start, int end) {
        this.start = start;
        this.end = end;
    }

    @Override
    protected Integer compute() {
        int sum = 0;
        //如果任务足够小就计算任务
        boolean canCompute = (end - start) <= threshold;
        if (canCompute) {
            for (int i = start; i <= end; i++) {
                sum += i;
            }
        } else {
            // 如果任务大于阈值，就分裂成两个子任务计算（分裂算法，可依情况调优）
            int middle = (start + end) / 2;
            ForkJoinTaskExample leftTask = new ForkJoinTaskExample(start, middle);
            ForkJoinTaskExample rightTask = new ForkJoinTaskExample(middle + 1, end);

            // 执行子任务
            leftTask.fork();
            rightTask.fork();

            // 等待任务执行结束合并其结果
            int leftResult = leftTask.join();
            int rightResult = rightTask.join();

            // 合并子任务
            sum = leftResult + rightResult;
        }
        return sum;
    }

    public static void main(String[] args) {
        ForkJoinPool forkjoinPool = new ForkJoinPool();

        //生成一个计算任务，计算1+2+3+4...100
        ForkJoinTaskExample task = new ForkJoinTaskExample(1, 100);

        //执行一个任务
        Future<Integer> result = forkjoinPool.submit(task);

        try {
            log.info("result:{}", result.get());
        } catch (Exception e) {
            log.error("exception", e);
        }
    }
}
</pre>

# BlockingQueue阻塞队列
主要应用场景：生产者消费者模型，是线程安全的 

![](https://img-blog.csdn.net/20180502175723970?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

## 阻塞情况：
1. 当队列满了进行入队操作 
2. 当队列空了的时候进行出队列操作

## 四套方法：
BlockingQueue提供了四套方法，分别来进行插入、移除、检查。每套方法在不能立刻执行时都有不同的反应。 

![](https://img-blog.csdn.net/2018050217591747?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

- Throws Exceptions ：如果不能立即执行就抛出异常。
- Special Value：如果不能立即执行就返回一个特殊的值。
- Blocks：如果不能立即执行就阻塞
- Times Out：如果不能立即执行就阻塞一段时间，如果过了设定时间还没有被执行，则返回一个值

## 实现类：
- ArrayBlockingQueue：它是一个有界的阻塞队列，内部实现是数组，初始化时指定容量大小，一旦指定大小就不能再变。采用FIFO方式存储元素。
- DelayQueue：阻塞内部元素，内部元素必须实现Delayed接口，Delayed接口又继承了Comparable接口，原因在于DelayQueue内部元素需要排序，一般情况按过期时间优先级排序。

<pre>
public interface Delayed extends Comparable<Delayed> {
    long getDelay(TimeUnit unit);
}
</pre>

- DalayQueue内部采用PriorityQueue与ReentrantLock实现。

<pre>
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
    implements BlockingQueue<E> {

    private final transient ReentrantLock lock = new ReentrantLock();
    private final PriorityQueue<E> q = new PriorityQueue<E>();
    ...
}
</pre>

- LinkedBlockingQueue：大小配置可选，如果初始化时指定了大小，那么它就是有边界的。不指定就无边界（最大整型值）。内部实现是链表，采用FIFO形式保存数据。

<pre>
public LinkedBlockingQueue() {
    this(Integer.MAX_VALUE);//不指定大小，无边界采用默认值，最大整型值
}
</pre>

- PriorityBlockingQueue:带优先级的阻塞队列。无边界队列，允许插入null。插入的对象必须实现Comparator接口，队列优先级的排序规则就是按照我们对Comparable接口的实现来指定的。我们可以从PriorityBlockingQueue中获取一个迭代器，但这个迭代器并不保证能按照优先级的顺序进行迭代。

<pre>
public boolean add(E e) {//添加方法
    return offer(e);
}
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    final ReentrantLock lock = this.lock;
    lock.lock();
    int n, cap;
    Object[] array;
    while ((n = size) >= (cap = (array = queue).length))
        tryGrow(array, cap);
    try {
        Comparator<? super E> cmp = comparator;//必须实现Comparator接口
        if (cmp == null)
            siftUpComparable(n, e, array);
        else
            siftUpUsingComparator(n, e, array, cmp);
        size = n + 1;
        notEmpty.signal();
    } finally {
        lock.unlock();
    }
    return true;
}
</pre>

- SynchronusQueue：只能插入一个元素，同步队列，无界非缓存队列，不存储元素。
