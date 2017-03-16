---
title: 多线程-Executors
date: 2016-12-19 11:11:03
tags: java_多线程
---
在https://busnoseat.github.io/2016/12/15/并发模式-多线程/  这篇文章里，分析了线程池ThreadPoolExecutor的工作模式。这篇文章稍微分析下Executors.
Executors是ThreadPoolExecutor的执行器，可以使用java定义好的几种线程池，其实这几种线程池共用了一个构造方法，只是入参不一样就有了不一样的线程池。

<h4>newFixedThreadPool:可重用固定线程池</h4>
<pre> 
<code>public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }</code>
</pre>
<!--more-->
corePoolSize=maxPoolSize,队列采用LinkedBlockingQueue。意思就是线程池开始工作后，先后创建corePoolsize的线程来执行任务,超出的放到LinkedBlockingQueue无界队列中等待线程来取任务并执行，队列的size固定为Integer.MAX_VALUE。超过的任务采用拒绝策略，默认的拒绝策略就是报异常。线程执行任务结束后不关闭，因为核心线程数就是当前运行的线程数。
<br>
<h4>newCachedThreadPool:可重用可变尺寸的线程池</h4>
<pre> 
<code>public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }</code>
</pre>
corePoolSize=0，而maxPoolSize=Integer.MAX_VALU，队列采用SynchronousQueue。SynchronousQueue暂时可以看成缓存为1的队列。意思就是线程池根据任务长度创建小于或等于同数目的线程来执行任务，最大的线程数为Integer.MAX_VALUE，超出的任务采用拒绝策略。每个线程执行完任务有60分的时间来接受新任务，所以总共启用的线程数要小于任务长度的。又因为核心线程数为0，所以线程超出等待时间后都会关闭。短时间的线程并发数很高，因为一个线程大概占到1M左右，1000个任务就是1G的内存，如果每个线程hold的时间长会造成内存泄漏。和newFixedThreadPool最大的区别就是后者线程数固定，其他任务可以缓存到size很大的队列中去，而前者就是启动size很大的线程来执行任务。
<br>
<h4>newSingleThreadExecutor:单任务线程池</h4>
<pre> 
<code> public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }</code>
</pre>
单任务线程池还用得着说么。。  不就是newFixedThreadPool的一种特例么，当核心线程数等于1的时候,就是单任务线程池了啊
<br>
<h4>newScheduledThreadPool:定时定长线程池</h4>
<pre> 
<code> public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
    }</code>
</pre>
和newFixedThreadPool定长线程池一样，区别是这个是定时任务性质的线程池，可以定时延迟多长时间后执行任务以及执行任务的频率。quartz的工作原理就是持有了一个ScheduledExecutorService的引用用于调度定时任务。
定时任务线程池请移步至 https://busnoseat.github.io/2016/12/29/多线程-ScheduledThreadPoolExecutor


