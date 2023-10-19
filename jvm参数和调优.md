---
title: jvm参数和调优
date: 2022-12-13 23:00:00
tags: jvm
---

jvm参数和调优
<!--more-->

## 堆设置
+ -Xms：设置jvm启动时堆内存的初始大小
+ -Xmx：设置堆内存最大值
+ -Xmn：设置年轻代大小，剩下的为老年代空间大小
+ -XX:SurvivorRatio
设置Eden区和Survivor区的空间比例 默认为8，即Eden区：From区：To区 = 8： 1：1
+ -XX:NewRatio 
设置新生代（Eden + 2*Survivor）与老年代（不包括永久区）的比值，默认值为2。 

## 收集器设置
+ -XX:+UseSerialGC 设置串行收集器
+ -XX:+UseParallelGC 设置并行收集器
+ -XX:+UseParalledlOldGC 设置并行年老代收集器
+ -XX:+UseConcMarkSweepGC 设置并发收集器

## 并行收集器设置
+ -XX:ParallelGCThreads=n 设置并行收集器收集时使用的CPU数。并行收集线程数。
+ -XX:MaxGCPauseMillis=n 设置并行收集最大暂停时间
+ -XX:GCTimeRatio=n 设置垃圾回收时间占程序运行时间的百分比。公式为1/(1+n)

## 并发收集器设置
-XX:+CMSIncrementalMode 设置为增量模式。适用于单CPU情况。
-XX:ParallelGCThreads=n 设置并发收集器年轻代收集方式为并行收集时，使用的CPU数。并行收集线程数。

## 垃圾回收统计信息
+ -XX:+PrintGC
+ -XX:+PrintGCDetails
+ -XX:+PrintGCTimeStamps
+ -Xloggc:filename

## 典型配置
```
java -Xmx3550m -Xms3550m -Xmn2g -Xss128k -XX:ParallelGCThreads=20 -XX:+UseConcMarkSweepGC -XX:+UseParNewGC

```
-XX:+UseConcMarkSweepGC设置年老代为并发收集。
-XX:+UseParNewGC 设置年轻代为并行收集，可与CMS收集同时使用。
-XX:ParallelGCThreads=20 配置并行收集器的线程数，即：同时多少个线程一起进行垃圾回收。此值最好配置与处理器数目相等。

```
-Xmx10240m -Xms10240m -Xmn2560m -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m -XX:+UseConcMarkSweepGC -XX:+UseCMSInitiatingOccupancyOnly -XX:+ExplicitGCInvokesConcurrent -XX:CMSInitiatingOccupancyFraction=75 -XX:ParallelGCThreads=4 -XX:CICompilerCount=4 -XX:+ExitOnOutOfMemoryError -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps/ -Dsun.net.inetaddr.ttl=1800

```



