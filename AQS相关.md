---
title: AQS相关
date: 2017-06-28 11:27:29
tags: java_多线程
---
 
 多线程并发情况下需要考虑线程安全问题，一般用锁来确保核心代码只有单个线程能运行。长用的有synchronized、ReentrantLock、ReadWriteLock，这些内部实现都依赖于AbstractQueuedSynchronizer类。

 # 大致流程
  1. 线程尝试获取锁，如果获取到，线程状态保持RUNNING,程序按照业务流程走下去
  2. 如果获取失败,说明已经有其他线程获取锁了,将此线程加入到等待队列中，并将该线程挂起
  3. 线程尝试释放锁，并将等待队列中的线程唤醒‘

<!--more-->

 # ReentrantLock
 
 <pre><code>
public class ReentrantLock implements Lock, java.io.Serializable {
    private final Sync sync;
    abstract static class Sync extends AbstractQueuedSynchronizer {
        abstract void lock();
        final boolean nonfairTryAcquire(int acquires) {... }
    //默认非公平锁
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    //fair为true时，采用公平锁策略
    public ReentrantLock(boolean fair) {    
        sync = fair ? new FairSync() : new NonfairSync();
    }
    public void lock() {
        sync.lock();
    }
    public void unlock() {    sync.release(1);}
    public Condition newCondition() {    
        return sync.newCondition();
    }
    ...
}
 </code></pre> 

 ReentrantLock里Sync继承自AQS，NonfairSynch和FairSync又继承Sync,所以公平锁和非公平锁都是AQS的实现类，主要实现AQS的tryAcquire()方法（尝试获取锁）。默认非公平锁。


 <pre><code>
     static final class NonfairSync extends Sync {
         final void lock() {
         	//尝试获取锁 ，将AQS的state从0变为1  state:成功获取锁并运行的线程总共获取了锁的次数  
         	//获取成功 将本线程记录 
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            //获取失败 直接调用AQS的acquire()方法 来进行线程入队操作
            acquire(1);
    }
}


    static final class FairSync extends Sync {
        final void lock() {
        	// 公平锁直接调用 
            acquire(1);
        }
 </code></pre>
 非公平锁中每次新线程执行lock方法的时候会先尝试获取锁，都会和等待队列中的首发线程一起争取锁，不排队是如此的不公平。公平锁直接调用AQS的acquire()方法。acquire方法里tryAcquire方法还是委托给子类实现



<pre><code>

   static final class NonfairSync extends Sync {
        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }

        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            // 0个线程持锁 再次尝试获取锁 体现不公平
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            // >=1 个持锁线程 这种是重入锁 state自增1  线程正常运行 不挂起 
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
        }


     static final class FairSync extends Sync {
           protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
             // 0个线程持锁 并且没有线程再等待队列里排队 才可以尝试获取锁 很礼貌 很公平 
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            // >=1 个持锁线程 这种是重入锁 state自增1  线程正常运行 不挂起 
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
</code></pre>
重入锁，即线程可以重复获取已经持有的锁。在非公平和公平锁中，都对重入锁进行了实现，一般用于递归获取锁的情况。
没有重入锁的时候 AQS的state是1 表明获取了该线程总共获取了一次，多次获取锁，state每次需要自增


# AQS (AbstractQueuedSynchronizer)

<pre><code>
	public abstract class AbstractQueuedSynchronizer extends
    AbstractOwnableSynchronizer implements java.io.Serializable { 
    //等待队列的头节点
    private transient volatile Node head;
    //等待队列的尾节点
    private transient volatile Node tail;
    //持有锁的线程数  0代表没有线程持有锁  1代表一个线程持有锁  >1代表该线程重复获取锁（重入锁）
    private volatile int state;
    protected final int getState() { return state;}
    protected final void setState(int newState) { state = newState;}
    ...

    static final class Node {
        static final Node SHARED = new Node();
        static final Node EXCLUSIVE = null;
        static final int CANCELLED =  1;
        static final int SIGNAL    = -1;
        static final int CONDITION = -2;
        static final int PROPAGATE = -3;
        volatile int waitStatus;
        volatile Node prev;
        volatile Node next;
        volatile Thread thread;
        Node nextWaiter;
        ...
    }

}
</code></pre>

AQS是用来建锁或其他同步组件的基础框架，上图列举了些重要变量。通过内置的FIFO队列Node来完成资源获取线程的排队工作

## node介绍以及node入队三步骤
![Image text](/asset/article/20170628/20170628001.jpg)
AQS的head指向node队列中的头结点，tail指向node队列中的尾节点，队列和队列之间通过prev和next来互相关联，形成类似蝴蝶扣的连接。如若新增节点
 1. 将新节点的prev指向原尾节点
 2. 将原尾节点的next指向新节点 形成蝴蝶扣
 3. 将AQS的tail指向新节点

需要额外注意的是 :头结点node1永远是个空节点（thread==null），头结点就是正在运行的线程。如果一个线程的prev是head头结点，说明头结点已经正在运行，可能执行结束，该线程可以再次尝试获取锁。如果获取成功，自己获取锁可以执行了，同时head后移指向自己，并把该节点node
的thread置为null
 <pre><code>
 	  public final void acquire(int arg) {
 	     //step1: 尝试获取锁 委托子类完成
 	     //step2: 获取锁失败，需要将该线程打包成一个node节点
 	     //step3：将新打包的节点加到队列中
 	     //step4: 入队成功后  挂起线程
 	     //step5：挂起的线程被唤醒后 返回线程挂起已被中断标志
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
      }
 </code></pre>
  
  ## addWaiter方法将线程打包成新节点
  <pre><code>
        //上步骤中传进来的Node.EXCLUSIVE为null
  	    private Node addWaiter(Node mode) {
  	    //新建一个新节点node 并初始化thread为当前线程
        Node node = new Node(Thread.currentThread(), mode);
        Node pred = tail;
        if (pred != null) {
        //如果存在尾节点 说明有队列 就需要新节点入队三步骤 并返回新节点
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        //如果不存在尾节点
        enq(node);
        return node;
    }


     private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { 
                //不存在尾节点 新建一个node并设置为头节点,尾节点  所以predecessor队列中的头结点head其实是一个空节点
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                //有了尾节点再把新节点入队三步骤
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
    </code></pre>


## acquireQueued方法 将新节点入队

<pre><code>
	    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                //在入队前  都要核实下是否还需要挂起线程  如果前节点为head就可以直接再次尝试获取锁
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    //如果获取成功 头结点已经运行结束 head后移指向该线程 并把该线程的thread置为null （该线程是头结点表明正在运行的线程）
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                //还是获取失败 就需要挂起线程了
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

      private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        //只有当前任线程状态为signal的时候才可以挂起线程
        if (ws == Node.SIGNAL)  
            return true;
        if (ws > 0) {
        //如果前任线程状态为已取消 将前任的前任和新节点组成蝴蝶扣
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
        //前任状态为signa以下 更新成signal
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }

        private final boolean parkAndCheckInterrupt() {
        //挂起线程 等待唤醒
        LockSupport.park(this);
        //唤醒后返回 线程挂起已被中断标志
        return Thread.interrupted();
    }

</code></pre>

总结： 以下图就是总结 
![Image text](/asset/article/20170628/20170628002.png)

感谢 http://www.jianshu.com/p/4358b1466ec9 收益匪浅



