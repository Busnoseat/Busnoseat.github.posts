---
title: java阻塞队列
date: 2019-12-11 17:56:47
tags: java_多线程
---

阻塞队列是一种队列。<!--more-->所以一方面是可以存储数据的,（在线程池中，当提交的任务不能被立即得到执行的时候，线程池就会将提交的任务放到一个阻塞的任务队列中来），同时它还有阻塞线程的作用（消费者生产者模式）。主要体现在以下
* 当队列中没有数据的情况下，消费者端的所有线程都会被自动阻塞（挂起），直到有数据放
入队列。
* 当队列中填满数据的情况下，生产者端的所有线程都会被自动阻塞（挂起），直到队列中有
空的位置，线程被自动唤醒。

### java中默认的阻塞队列
* ArrayBlockingQueue ：由数组结构组成的有界阻塞队列。
* LinkedBlockingQueue ：由链表结构组成的有界阻塞队列。
* PriorityBlockingQueue ：支持优先级排序的无界阻塞队列。
* DelayQueue：使用优先级队列实现的无界阻塞队列。
* SynchronousQueue：不存储元素的阻塞队列。
* LinkedTransferQueue：由链表结构组成的无界阻塞队列。
* LinkedBlockingDeque：由链表结构组成的双向阻塞队列

### 阻塞队列的主要方法
![Image text](/asset/article/20191211/1.png)

### ArrayBlockingQueue
```
...
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {
    /**数组 用来存放数据*/
    final Object[] items;

    /**取得下标*/
    int takeIndex;

    /**存的下标*/
    int putIndex;

    /**队列里已存在的size*/
    int count;

    /**锁*/
    final ReentrantLock lock;

    /**有数据可以取的状态*/
    private final Condition notEmpty;

    /**数据可以存放的状态*/
    private final Condition notFull;

    /**初始化队列大小,有界*/
    public ArrayBlockingQueue(int capacity) ;

    /**初始化队列大小的同时，指定锁是公平锁还是非公平锁*/
    public ArrayBlockingQueue(int capacity, boolean fair)

    ...
        }
```
由以上成员变量和构造方法可知：ArrayBlockingQueue用数组实现的有界阻塞队列。此队列按照先进先出（FIFO）的原则对元素进行排序。默认情况下
不保证访问者公平的访问队列，即先阻塞的生产者线程，可以先往队列里插入元素，先阻塞的消费者线程，可以先从队列里获取元素。

```

    /**存放数据的核心方法*/
    private void enqueue(E x) {
        // assert lock.getHoldCount() == 1;
        // assert items[putIndex] == null;
        final Object[] items = this.items;
        /**赋值*/
        items[putIndex] = x;
        /**存完后下标自增1，下个线程存放数据的时候可以直接操作下标 如果下标自增后达到了数组最大长度，则下标直接从0开始*/
        if (++putIndex == items.length)
            putIndex = 0;
        count++;
        /**存完后数组状态是不空，所以要唤醒可能存在的获取队列*/
        notEmpty.signal();
    } 

    /**获取数据的核心方法*/
    private E dequeue() {
        // assert lock.getHoldCount() == 1;
        // assert items[takeIndex] != null;
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        E x = (E) items[takeIndex];
        /**取完后数组对应下标赋值null helpGC */
        items[takeIndex] = null;
        /**取完后下标自增1，下个线程取数据的时候可以直接操作下标 如果下标自增后达到了数组最大长度，则下标从0开始 */
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
        notFull.signal();
        return x;
    }   
```
需要注意的是数组使用了两个游标变量：takeIndex和putIndex，配合这两个变量之后数组的使用就像是一个环形队列一样了。并且每次存或取得时候都会判断，如果数组里的数据达到了capacity,就不再存取，即putIndex存一圈后又从0开始时永远不会到达takeIndex所在位置。
```
    /**add(E):默认使用父类的add方法  就是调用offer方法 如果失败了则直接抛错*/
    public boolean add(E e) {
        if (offer(e))
            return true;
        else
            throw new IllegalStateException("Queue full");
    }
 
   /**offer(E):直接调用核心方法enqueue 成功返回true 失败返回false*/
     public boolean offer(E e) {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
        	/**如果数组里的数据达到了capacity,就不再存取*/
            if (count == items.length)
                return false;
            else {
                enqueue(e);
                return true;
            }
        } finally {
            lock.unlock();
        }
    }

   /**offer(E,Long,TimeUnit): 如果数组里的数据达到了capacity 则阻塞指定时间后再存放，成功返回true 失败返回false*/
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

        checkNotNull(e);
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length) {
                if (nanos <= 0)
                    return false;
                nanos = notFull.awaitNanos(nanos);
            }
            enqueue(e);
            return true;
        } finally {
            lock.unlock();
        }
    } 

    /**put(E): 如果数组里的数据达到了capacity 则一直阻塞，直到状态为非满（唤醒）后再存放，成功返回true 失败返回false*/
    public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length)
                notFull.await();
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }  
```
以上就是add、offer、put方法差异的原因。

### LinkedBlockingQueue
```
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {

    /**使用链表来存储数据 只有成员变量next没有pre，就是只可以从前往后*/
    static class Node<E> {
        E item;

        Node<E> next;

        Node(E x) { item = x; }
    }

    /**链表存储数据的容量*/
    private final int capacity;

    /**链表已经存储的size*/
    private final AtomicInteger count = new AtomicInteger();

    /**头结点 头结点默认是一个空节点*/
    transient Node<E> head;

    /**尾结点*/
    private transient Node<E> last;

    /**取数据的线程专用锁*/
    private final ReentrantLock takeLock = new ReentrantLock();

    /**有数据可以取的状态*/
    private final Condition notEmpty = takeLock.newCondition();

    /**存放数据的线程专用锁*/
    private final ReentrantLock putLock = new ReentrantLock();

    /**有数据可以存的状态*/
    private final Condition notFull = putLock.newCondition();

    /**默认最大容量为Integer最大值*/
    public LinkedBlockingQueue(){ this(Integer.MAX_VALUE); }

    /**指定容量初始化*/
    public LinkedBlockingQueue(int capacity)

    }

```
由以上成员变量和构造方法可知：LinkedBlockingQueue使用链表来作为队列的数据结构，存和取的操作使用了不同的锁

```
    /**存数据的核心方法： 将尾节点的next节点指向新增节点*/
    private void enqueue(Node<E> node) {
        // assert putLock.isHeldByCurrentThread();
        // assert last.next == null;
        last = last.next = node;
    }
    
    /**取数据的核心方法：头结点为空，取出头结点的后一个节点返回 */
    private E dequeue() {
        // assert takeLock.isHeldByCurrentThread();
        // assert head.item == null;
        Node<E> h = head;
        Node<E> first = h.next;
        h.next = h; // help GC
        /**返回first节点前 将first设置为头结点 并且将头结点设为null*/
        head = first;
        E x = first.item;
        first.item = null;
        return x;
    }
```
取节点稍微复杂点，用图来描述下：
![Image text](/asset/article/20191211/2.png)
代码里的实现并不是直接将first节点拿掉然后将head节点的next指向second节点，而是直接将first设置为头结点 并且将头结点item值设为null，这样first节点就转变成了空节点。

```
    /**add方法： 还是调用父类的方法 不在阐述*/

    /**offer(E):直接调用核心方法enqueue 成功返回true 失败返回false*/
    public boolean offer(E e) {
        if (e == null) throw new NullPointerException();
        final AtomicInteger count = this.count;
        /**如果链表满了 不能再继续存数据 直接返回false*/
        if (count.get() == capacity)
            return false;
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        putLock.lock();
        try {
            if (count.get() < capacity) {

                enqueue(node);
                /**存完数据后 count自增1 */
                c = count.getAndIncrement();
                /**如果当前数据容量还可以继续存 则唤醒等待存放的线程*/
                if (c + 1 < capacity)
                    notFull.signal();
            }
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
        return c >= 0;
    }


   /**offer(E,Long,TimeUnit): 如果数组里的数据达到了capacity 则阻塞指定时间后再存放，成功返回true 失败返回false*/
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

        if (e == null) throw new NullPointerException();
        long nanos = unit.toNanos(timeout);
        int c = -1;
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            while (count.get() == capacity) {
                if (nanos <= 0)
                    return false;
                nanos = notFull.awaitNanos(nanos);
            }
            enqueue(new Node<E>(e));
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
        return true;
    }

    /**put(E): 如果数组里的数据达到了capacity 则一直阻塞，直到状态为非满（唤醒）后再存放，成功返回true 失败返回false*/
    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        // Note: convention in all put/take/etc is to preset local var
        // holding count negative to indicate failure unless set.
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            /*
             * Note that count is used in wait guard even though it is
             * not protected by lock. This works because count can
             * only decrease at this point (all other puts are shut
             * out by lock), and we (or some other waiting put) are
             * signalled if it ever changes from capacity. Similarly
             * for all other uses of count in other wait guards.
             */
            while (count.get() == capacity) {
                notFull.await();
            }
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
    }
```
通过以上代码可知，add、offer、put的差异就在于调用enqueue前是否等待以及等待时长。需要注意的是每次存完数据后都有notFull.signal()这段代码，这是因为LinkedBlockingQueue采用了双锁的机制，导致存完数据后不单要唤醒取线程，还要唤醒可能存在的存线程。如下图所示，ArrayBlockingQueue的put线程从acquireLock到releaseLock这阶段不会有存或者取线程干扰。LinkedBlockingQueue的put线程会阻塞后续的put线程，但是不会阻塞take线程，这就导致一种可能：put线程发现队列满了，在准备阻塞到阻塞这段时间让出了cpu碎片并且完成了2次take操作，虽然2次take都会唤醒put线程，但是此时put线程还没阻塞。所以这就需要后续的put线程或者take线程取唤醒了。
![Image text](/asset/article/20191211/3.png)


感谢文章：https://www.jianshu.com/p/4028efdbfc35
