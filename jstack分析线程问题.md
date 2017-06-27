---
title: jstack分析线程问题
date: 2017-05-31 14:27:16
tags: jvm
---
原文地址：http://www.jianshu.com/p/6690f7e92f27  感谢占小狼！！

如何查看测试或线上 占用cpu较高的线程？

# step1 : top       
查询所有的进程占用
 <!--more--> 
![Image text](/asset/article/20170531/1.png)

# step2 : top -Hp pid
查询该进程所有的线程占用
![Image text](/asset/article/20170531/2.png)

# step3 : jstack pid
查看该线程的堆栈状态
![Image text](/asset/article/20170531/3.png)


# 如何分析堆栈信息

在thread dump中每个线程都有一个nid，对应的是16进制的线程，观察该线程的状态和执行方法，隔段时间jstack查询下
如果该nid一直在执行同一个方法，就可以证明是否递归或死循环。
如果该nid的状态一直是waiting或者blocking，就要考虑线程死锁问题。

# 实例1：多线程竞争synchronized锁
![Image text](/asset/article/20170531/4.png)

很明显：线程1获取到锁，处于RUNNABLE状态，线程2处于BLOCK状态
1：locked <0x000000076bf62208>说明线程1对地址为0x000000076bf62208对象进行了加锁；
2：waiting to lock <0x000000076bf62208> 说明线程2在等待地址为0x000000076bf62208对象上的锁；
3：waiting for monitor entry [0x000000001e21f000]说明线程1是通过synchronized关键字进入了监视器的临界区，并处于"Entry Set"队列，等待monitor
具体实现可以参考 http://www.jianshu.com/p/c5058b6fe8e5；

# 实例2：通过wait挂起线程
![Image text](/asset/article/20170531/5.png)

线程1和2都处于WAITING状态
1：线程1和2都是先locked <0x000000076bf62500>，再waiting on <0x000000076bf62500>，之所以先锁再等同一个对象，是因为wait方法需要先通过synchronized获得该地址对象的monitor；
2：waiting on <0x000000076bf62500>说明线程执行了wait方法之后，释放了monitor，进入到"Wait Set"队列，等待其它线程执行地址为0x000000076bf62500对象的notify方法，并唤醒自己，
具体实现可以参考 http://www.jianshu.com/p/f4454164c017；
