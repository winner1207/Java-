# 概述
Java并发容器JUC是三个单词的缩写。是JDK下面的一个包名。即Java.util.concurrency。 上一节介绍了ArrayList、HashMap、HashSet对应的同步容器保证其线程安全，这节我们介绍一下其对应的并发容器。

## ArrayList –> CopyOnWriteArrayList

示例代码：[CopyOnWriteArrayListExample.java](../src/main/java/com/mmall/concurrency/example/concurrent/CopyOnWriteArrayListExample.java)

CopyOnWriteArrayList 写操作时复制，当有新元素添加到集合中时，从原有的数组中拷贝一份出来，然后在新的数组上作写操作，将原来的数组指向新的数组。整个数组的add操作都是在锁的保护下进行的，防止并发时复制多份副本。读操作是在原数组中进行，不需要加锁

- 缺点：  
1.写操作时复制消耗内存，如果元素比较多时候，容易导致young gc 和full gc。  
2.不能用于实时读的场景.由于复制和add操作等需要时间，故读取时可能读到旧值。能做到最终一致性，但无法满足实时性的要求，更适合读多写少的场景。如果无法知道数组有多大，或者add,set操作有多少，慎用此类，在大量的复制副本的过程中很容易出错。

- 设计思想：  
1.读写分离  
2.最终一致性  
3.使用时另外开辟空间，防止并发冲突

- 源码分析

<pre>
//构造方法
public CopyOnWriteArrayList(Collection<? extends E> c) {
    Object[] elements;//使用对象数组来承载数据
    if (c.getClass() == CopyOnWriteArrayList.class)
        elements = ((CopyOnWriteArrayList<?>)c).getArray();
    else {
        elements = c.toArray();
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elements.getClass() != Object[].class)
            elements = Arrays.copyOf(elements, elements.length, Object[].class);
    }
    setArray(elements);
}

//添加数据方法
public boolean add(E e) {
    final ReentrantLock lock = this.lock;//使用重入锁，保证线程安全
    lock.lock();
    try {
        Object[] elements = getArray();//获取当前数组数据
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);//复制当前数组并且扩容+1
        newElements[len] = e;//将要添加的数据放入新数组
        setArray(newElements);//将原来的数组指向新的数组
        return true;
    } finally {
        lock.unlock();
    }
}

//获取数据方法，与普通的get没什么差别
private E get(Object[] a, int index) {
    return (E) a[index];
}
</pre>


## HashSet –> CopyOnWriteArraySet

示例代码：[CopyOnWriteArraySetExample.java](../src/main/java/com/mmall/concurrency/example/concurrent/CopyOnWriteArraySetExample.java)

- 它是线程安全的，底层实现使用的是CopyOnWriteArrayList，因此它也适用于大小很小的set集合，只读操作远大于可变操作。因为他需要copy整个数组，所以包括add、remove、set它的开销相对于大一些。
- 迭代器不支持可变的remove操作。使用迭代器遍历的时候速度很快，而且不会与其他线程发生冲突。
- 源码分析：

<pre>
//构造方法
public CopyOnWriteArraySet() {
    al = new CopyOnWriteArrayList<E>();//底层使用CopyOnWriteArrayList
}

//添加元素方法，基本实现原理与CopyOnWriteArrayList相同
private boolean addIfAbsent(E e, Object[] snapshot) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] current = getArray();
        int len = current.length;
        if (snapshot != current) {//添加了元素去重操作
            // Optimize for lost race to another addXXX operation
            int common = Math.min(snapshot.length, len);
            for (int i = 0; i < common; i++)
                if (current[i] != snapshot[i] && eq(e, current[i]))
                    return false;
            if (indexOf(e, current, common, len) >= 0)
                    return false;
        }
        Object[] newElements = Arrays.copyOf(current, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
</pre>

## TreeSet –> ConcurrentSkipListSet

示例代码：[ConcurrentSkipListSetExample.java](../src/main/java/com/mmall/concurrency/example/concurrent/ConcurrentSkipListSetExample.java)

它是JDK6新增的类，同TreeSet一样支持自然排序，并且可以在构造的时候自己定义比较器。

- 同其他set集合，是基于map集合的（基于ConcurrentSkipListMap），在多线程环境下，里面的contains、add、remove操作都是线程安全的。
- 多个线程可以安全的并发的执行插入、移除、和访问操作。但是对于批量操作addAll、removeAll、retainAll和containsAll并不能保证以原子方式执行，原因是addAll、removeAll、retainAll底层调用的还是contains、add、remove方法，只能保证每一次的执行是原子性的，代表在单一执行操纵时不会被打断，但是不能保证每一次批量操作都不会被打断。在使用批量操作时，还是需要手动加上同步操作的。
- 不允许使用null元素的，它无法可靠的将参数及返回值与不存在的元素区分开来。
- 源码分析：

<pre>
//构造方法
public ConcurrentSkipListSet() {
    m = new ConcurrentSkipListMap<E,Object>();//使用ConcurrentSkipListMap实现
}
</pre>

## HashMap –> ConcurrentHashMap

示例代码：[ConcurrentHashMapExample.java](../src/main/java/com/mmall/concurrency/example/concurrent/ConcurrentHashMapExample.java)

- 不允许空值，在实际的应用中除了少数的插入操作和删除操作外，绝大多数我们使用map都是读取操作。而且读操作大多数都是成功的。基于这个前提，它针对读操作做了大量的优化。因此这个类在高并发环境下有特别好的表现。
- ConcurrentHashMap作为Concurrent一族，其有着高效地并发操作，相比Hashtable的笨重，ConcurrentHashMap则更胜一筹了。
- 在1.8版本以前，ConcurrentHashMap采用分段锁的概念，使锁更加细化，但是1.8已经改变了这种思路，而是利用CAS+Synchronized来保证并发更新的安全，当然底层采用数组+链表+红黑树的存储结构。
- 源码分析：推荐参考博文：[J.U.C之Java并发容器：ConcurrentHashMap](https://blog.csdn.net/chenssy/article/details/73521950)

## TreeMap –> ConcurrentSkipListMap

示例代码：[ConcurrentSkipListMapExample.java](../src/main/java/com/mmall/concurrency/example/concurrent/ConcurrentSkipListMapExample.java)

- 底层实现采用SkipList跳表
- 用ConcurrentHashMap与ConcurrentSkipListMap做性能测试，在4个线程1.6W的数据条件下，前者的数据存取速度是后者的4倍左右。但是后者有几个前者不能比拟的优点： 
1、Key是有序的 
2、支持更高的并发，存储时间与线程数无关

## 安全共享对象策略

- 线程限制：一个被线程限制的对象，由线程独占，并且只能被占有它的线程修改
- 共享只读：一个共享只读的对象，在没有额外同步的情况下，可以被多个线程并发访问，但是任何线程都不能修改它
- 线程安全对象：一个线程安全的对象或者容器，在内部通过同步机制来保障线程安全，所以其他线程无需额外的同步就可以通过公共接口随意访问他
- 被守护对象：被守护对象只能通过获取特定的锁来访问。


