# new Thread的弊端
- 每次new Thread 新建对象，性能差
- 线程缺乏统一管理，可能无限制的新建线程，相互竞争，可能占用过多的系统资源导致死机或者OOM（out of memory 内存溢出），这种问题的原因不是因为单纯的new一个Thread，而是可能因为程序的bug或者设计上的缺陷导致不断new Thread造成的。
- 缺少更多功能，如更多执行、定期执行、线程中断。

# 线程池的好处
- 重用存在的线程，减少对象创建、消亡的开销，性能好
- 可有效控制最大并发线程数，提高系统资源利用率，同时可以避免过多资源竞争，避免阻塞。
- 提供定时执行、定期执行、单线程、并发数控制等功能。

# 线程池类图
![](https://img-blog.csdn.net/20180505135807405?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

在线程池的类图中，我们最常使用的是最下边的Executors,用它来创建线程池使用线程。那么在上边的类图中，包含了一个Executor框架，它是一个根据一组执行策略的调用调度执行和控制异步任务的框架，目的是提供一种将任务提交与任务如何运行分离开的机制。它包含了三个executor接口：

- Executor:运行新任务的简单接口
- ExecutorService：扩展了Executor，添加了用来管理执行器生命周期和任务生命周期的方法
- ScheduleExcutorService：扩展了ExecutorService，支持Future和定期执行任务

# 线程池核心类-ThreadPoolExecutor
参数说明：ThreadPoolExecutor一共有七个参数，这七个参数配合起来，构成了线程池强大的功能。

- corePoolSize：核心线程数量
- maximumPoolSize：线程最大线程数
- workQueue：阻塞队列，存储等待执行的任务，很重要，会对线程池运行过程产生重大影响

>
当我们提交一个新的任务到线程池，线程池会根据当前池中正在运行的线程数量来决定该任务的处理方式。处理方式有三种： 
1、直接切换（SynchronusQueue） 
2、无界队列（LinkedBlockingQueue）能够创建的最大线程数为corePoolSize,这时maximumPoolSize就不会起作用了。当线程池中所有的核心线程都是运行状态的时候，新的任务提交就会放入等待队列中。 
3、有界队列（ArrayBlockingQueue）最大maximumPoolSize，能够降低资源消耗，但是这种方式使得线程池对线程调度变的更困难。因为线程池与队列容量都是有限的。所以想让线程池的吞吐率和处理任务达到一个合理的范围，又想使我们的线程调度相对简单，并且还尽可能降低资源的消耗，我们就需要合理的限制这两个数量 
分配技巧： [如果想降低资源的消耗包括降低cpu使用率、操作系统资源的消耗、上下文切换的开销等等，可以设置一个较大的队列容量和较小的线程池容量，这样会降低线程池的吞吐量。如果我们提交的任务经常发生阻塞，我们可以调整maximumPoolSize。如果我们的队列容量较小，我们需要把线程池大小设置的大一些，这样cpu的使用率相对来说会高一些。但是如果线程池的容量设置的过大，提高任务的数量过多的时候，并发量会增加，那么线程之间的调度就是一个需要考虑的问题。这样反而可能会降低处理任务的吞吐量。]

- keepAliveTime：线程没有任务执行时最多保持多久时间终止（当线程中的线程数量大于corePoolSize的时候，如果这时没有新的任务提交核心线程外的线程不会立即销毁，而是等待，直到超过keepAliveTime）
- unit：keepAliveTime的时间单位
- threadFactory：线程工厂，用来创建线程，有一个默认的工场来创建线程，这样新创建出来的线程有相同的优先级，是非守护线程、设置好了名称）
- rejectHandler：当拒绝处理任务时(阻塞队列满)的策略（AbortPolicy默认策略直接抛出异常、CallerRunsPolicy用调用者所在的线程执行任务、DiscardOldestPolicy丢弃队列中最靠前的任务并执行当前任务、DiscardPolicy直接丢弃当前任务）

![](https://img-blog.csdn.net/20180505142542206?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

>
corePoolSize、maximumPoolSize、workQueue 三者关系：如果运行的线程数小于corePoolSize的时候，直接创建新线程来处理任务。即使线程池中的其他线程是空闲的。如果运行中的线程数大于corePoolSize且小于maximumPoolSize时，那么只有当workQueue满的时候才创建新的线程去处理任务。如果corePoolSize与maximumPoolSize是相同的，那么创建的线程池大小是固定的。这时有新任务提交，当workQueue未满时，就把请求放入workQueue中。等待空线程从workQueue取出任务。如果workQueue此时也满了，那么就使用另外的拒绝策略参数去执行拒绝策略。

初始化方法：由七个参数组合成四个初始化方法 
![](https://img-blog.csdn.net/20180505141614527?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

其他方法：

| 序号 | 方法名 | 描述 |
| 1	| execute()	| 提交任务，交给线程池执行 |
| 2	| submit()	| 提交任务，能够返回执行结果 execute+Future |
| 3	| shutdown() | 关闭线程池，等待任务都执行完 |
| 4	| shutdownNow() | 关闭线程池，不等待任务执行完 |
| 5	| getTaskCount() | 线程池已执行和未执行的任务总数 |
| 6 | getCompleteTaskCount() | 已完成的任务数量 |
| 7	| getPoolSize() | 线程池当前的线程数量 |
| 8	| getActiveCount() | 当前线程池中正在执行任务的线程数量 |

线程池生命周期： 
![](https://img-blog.csdn.net/201805051421294?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

- running：能接受新提交的任务，也能处理阻塞队列中的任务
- shutdown：不能处理新的任务，但是能继续处理阻塞队列中任务
- stop：不能接收新的任务，也不处理队列中的任务
- tidying：如果所有的任务都已经终止了，这时有效线程数为0
- terminated：最终状态

# 使用Executor创建线程池
使用Executor可以创建四种线程池：分别对应上边提到的四种线程池初始化方法

## 1、Executors.newCachedThreadPool 
创建一个可缓存的线程池，如果线程池的长度超过了处理的需要，可以灵活回收空闲线程。如果没有可回收的就新建线程。

<pre>
//源码：
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
//使用方法：
public static void main(String[] args) {
    ExecutorService executorService = Executors.newCachedThreadPool();
    for (int i = 0; i < 10; i++) {
        final int index = i;
        executorService.execute(new Runnable() {
            @Override
            public void run() {
                log.info("task:{}", index);
            }
        });
    }
    executorService.shutdown();
}
</pre>

值得注意的一点是，newCachedThreadPool的返回值是ExecutorService类型，该类型只包含基础的线程池方法，但却不包含线程监控相关方法，因此在使用返回值为ExecutorService的线程池类型创建新线程时要考虑到具体情况。 

![](https://img-blog.csdn.net/20180505143434499?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

## 2、newFixedThreadPool 
定长线程池，可以线程现成的最大并发数，超出在队列等待

<pre>
//源码：
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
//使用方法：
public static void main(String[] args) {
    ExecutorService executorService = Executors.newFixedThreadPool(3);
    for (int i = 0; i < 10; i++) {
        final int index = i;
        executorService.execute(new Runnable() {
            @Override
            public void run() {
                log.info("task:{}", index);
            }
        });
    }
    executorService.shutdown();
}	
</pre>

## 3、newSingleThreadExecutor 
单线程化的线程池，用唯一的一个共用线程执行任务，保证所有任务按指定顺序执行（FIFO、优先级…）

<pre>
//源码
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
//使用方法：
public static void main(String[] args) {
    ExecutorService executorService = Executors.newSingleThreadExecutor();
    for (int i = 0; i < 10; i++) {
        final int index = i;
        executorService.execute(new Runnable() {
            @Override
            public void run() {
                log.info("task:{}", index);
            }
        });
    }
    executorService.shutdown();
}
</pre>

## 4、newScheduledThreadPool 
定长线程池，支持定时和周期任务执行

<pre>
//源码：
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,//此处super指的是ThreadPoolExecutor
          new DelayedWorkQueue());
}
//基础使用方法：
public static void main(String[] args) {
    ScheduledExecutorService executorService = Executors.newScheduledThreadPool(1);
    executorService.schedule(new Runnable() {
        @Override
        public void run() {
            log.warn("schedule run");
        }
    }, 3, TimeUnit.SECONDS);//延迟3秒执行
    executorService.shutdown();
}
</pre>

ScheduledExecutorService提供了三种方法可以使用： 

![](https://img-blog.csdn.net/20180505144730122?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

scheduleAtFixedRate：以指定的速率执行任务 
scheduleWithFixedDelay：以指定的延迟执行任务 
举例：

<pre>
executorService.scheduleAtFixedRate(new Runnable() {
    @Override
    public void run() {
        log.warn("schedule run");
    }
}, 1, 3, TimeUnit.SECONDS);//延迟一秒后每隔3秒执行
</pre>

小扩展：延迟执行任务的操作，java中还有Timer类同样可以实现

<pre>
Timer timer = new Timer();
timer.schedule(new TimerTask() {
    @Override
    public void run() {
        log.warn("timer run");
    }
}, new Date(), 5 * 1000);
</pre>