# 概述
在上一节我们简略提到了ConcurrentHashMap是HashMap的线程安全类，那么这两个类的具体实现是怎样的呢？我们来了解一下。

# HashMap
## （1）初始化方法
HashMap的实现方式是：数组+链表 的形式。 

![](https://img-blog.csdn.net/20180417182317205?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

在HashMap中有两个参数会影响HashMap的性能：初始容量/加载因子

> 初始容量：Hash表中桶的数量 
> 加载因子：是Hash表在自动增加之前可以达到多满的一个尺度。

HashMap在类中定义了这两个参数:

<pre>
//初始容量，默认16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; 
//加载因子，默认0.75
static final float DEFAULT_LOAD_FACTOR = 0.75f;
</pre>

这两个参数的作用是：当Hash表中的条目数量超过了加载因子与当前容量的乘积，将会调用resize()进行扩容，将容量翻倍。 
这两个参数在初始化HashMap的时候可以进行设置：可以单独指定初始容量，也可以同时设置 

![](https://img-blog.csdn.net/20180417182851565?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

## （2）寻址方式
对于一个新插入的数据或者要读取的数据，HashMap将key按一定规则计算出hash值，并对数组长度进行取模结果作为在数组中查找的index。由于在计算机中取模的代价远远高于位操作的代价，因此HashMap要求数组的长度必须为2的N次方。此时它将key的hash值对2的n-1次方进行与运算，等同于取模运算。HashMap并不要求用户一定要设置一个2的N次方的初始化大小，它本身内部会通过运算（tableSizeFor方法）确定一个合理的符合2的N次方的大小去设置。

<pre>
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
</pre>

## （3）HashMap的线程不安全原因一：死循环
原因在于HashMap在多线程情况下，执行resize()进行扩容时容易造成死循环。 
扩容思路为它要创建一个大小为原来两倍的数组，保证新的容量仍为2的N次方，从而保证上述寻址方式仍然适用。扩容后将原来的数组从新插入到新的数组中。这个过程称为reHash。

【单线程下的reHash】 

![](https://img-blog.csdn.net/20180417184428269?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

- 扩容前：我们的HashMap初始容量为2，加载因子为1，需要向其中存入3个key，分别为5、9、11，放入第三个元素11的时候就涉及到了扩容。
- 第一步：先创建一个二倍大小的数组，接下来把原来数组中的元素reHash到新的数组中，5插入新的数组，没有问题。
- 第二步：将9插入到新的数组中，经过Hash计算，插入到5的后面。
- 第三步：将11经过Hash插入到index为3的数组节点中。

单线程reHash完全没有问题。

【多线程下的reHash】 

![](https://img-blog.csdn.net/20180417194252382?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

我们假设有两个线程同时执行了put操作，并同时触发了reHash的操作，图示的上层的线程1，下层是线程2。

- 线程1某一时刻执行完扩容，准备将key为5的元素的next指针指向9，由于线程调度分配的时间片被用完而停在了这一步操作
- 线程2在这一刻执行reHash操作并执行完数据迁移的整个操作。

接下来线程1被唤醒继续操作。 

![](https://img-blog.csdn.net/2018041719514289?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

- 执行上一轮的剩余部分，在处理key为5的元素时，将此key放在我们线程1申请的数组的索引1位置的链表的首部。理想状态是（线程1数组索引1）—> (Key=5) —> null

![](https://img-blog.csdn.net/20180418102150675?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

- 接着处理Key为9的元素，将key为9的元素插入在（索引1）与（key=5）之间，理想状态：（线程1数组索引1）—> （Key=9）—> （Key=5）—>null 

![](https://img-blog.csdn.net/20180418102812207?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

- 但是在处理完key为9的元素之后按理说应该结束了，但是由于线程2已经处理过了key=9与key=5的元素，即真实情况为（线程2数组索引1 —>（key=9）—> （key=5）—> null）|（线程1数组索引1 —> (key=9)—> （key=5）—> null），这时让线程1误以为key=9后面的key=5是从原数组还没有进行数组迁移的，接着又处理key=5。尝试将key=5放在k=9的前边，所以key=9与key=5之间就出现了一个循环。不断的被处理，交换顺序。

- key = 11的元素是无法插入到新数组中的。一旦我们去从新的数组中获取值得时候，就会出现死循环。

## （4）HashMap的线程不安全原因二：fail-fast
如果在使用迭代器的过程中有其他线程修改了map，那么将抛出ConcurrentModificationException，这就是所谓fail-fast。 
在每一次对HashMap进行修改的时候，都会变动类中的modCount域，即modCount变量的值。源码中是这样实现的：

<pre>
abstract class HashIterator {
        ...
        int expectedModCount;  // for fast-fail
        int index;             // current slot

        HashIterator() {
            expectedModCount = modCount;
            Node<K,V>[] t = table;
            current = next = null;
            index = 0;
            if (t != null && size > 0) { // advance to first entry
                do {} while (index < t.length && (next = t[index++]) == null);
            }
        }
        ...
}
</pre>

在每次迭代的过程中，都会判断modCount跟expectedModCount是否相等，如果不相等代表有人修改HashMap。源码：

<pre>
final Node<K,V> nextNode() {
    Node<K,V>[] t;
    Node<K,V> e = next;
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
    if (e == null)
        throw new NoSuchElementException();
    if ((next = (current = e).next) == null && (t = table) != null) {
        do {} while (index < t.length && (next = t[index++]) == null);
    }
    return e;
}
</pre>

解决办法：可以使用Collections的synchronizedMap方法构造一个同步的map，或者直接使用线程安全的ConcurrentHashMap来保证不会出现fail-fast策略。


# ConcurrentHashMap

## （1）结构 [Java7与Java8不同]
![](https://img-blog.csdn.net/20180418110431880?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

- Java7里面的ConcurrentHashMap的底层结构仍然是数组和链表，与HashMap不同的是ConcurrentHashMap的最外层不是一个大的数组，而是一个Segment数组。每个Segment包含一个与HashMap结构差不多的链表数组。
- 当我们读取某个Key的时候它先取出key的Hash值，并将Hash值得高sshift位与Segment的个数取模，决定key属于哪个Segment。接着像HashMap一样操作Segment。
- 为了保证不同的Hash值保存到不同的Segment中，ConcurrentHashMap对Hash值也做了专门的优化。
- Segment继承自J.U.C里的ReetrantLock，所以可以很方便的对Segment进行上锁。即分段锁。

![](https://img-blog.csdn.net/20180418111414251?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2plc29uam9rZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

- Java8废弃了Java7中ConcurrentHashMap中分段锁的方案，并且不使用Segment，转为使用大的数组。同时为了提高Hash碰撞下的寻址做了性能优化。

- Java8在列表的长度超过了一定的值（默认8）时，将链表转为红黑树实现。寻址的复杂度从O(n)转换为Olog(n)。


# 对比
- HashMap非线程安全、ConcurrentHashMap线程安全
- HashMap允许Key与Value为空，ConcurrentHashMap不允许
- HashMap不允许通过迭代器遍历的同时修改，ConcurrentHashMap允许。并且更新可见
