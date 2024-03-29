---
title: jvm运行时内存
date: 2022-11-24 17:00:00
tags: jvm
---

线程共享区域包括了【JAVA堆、方法区/永久代】。Java堆从GC的角度还可以细分为: 新生代(Eden区、From Survivor区和To Survivor区)和老年代。
<!--more-->
![Image text](/asset/article/20221123/4.png)
### 新生代
主要是用来存放新生的对象。一般占据堆的1/3空间。由于频繁创建对象，所以新生代会频繁触发MinorGC进行垃圾回收。
新生代又分为 Eden区、ServivorFrom、ServivorTo三个区。
* Eden区：Java新对象的出生地（如果新创建的对象占用内存很大，则直接分配到老年代）。当Eden区内存不够的时候就会触发MinorGC，对新生代区进行一次垃圾回收。
* ServivorTo：保留了一次MinorGC过程中的幸存者。
* ServivorFrom：上一次GC的幸存者，作为这一次GC的被扫描者

### 老年代
主要存放应用程序中生命周期长的内存对象。
老年代的对象比较稳定，所以MajorGC不会频繁执行。在进行MajorGC前一般都先进行了一次MinorGC，使得有新生代的对象晋身入老年代，导致空间不够用时才触发。当无法找到足够大的连续空间分配给新创建的较大对象时也会提前触发一次MajorGC进行垃圾回收腾出空间。
MajorGC采用标记—清除算法：首先扫描一次所有老年代，标记出存活的对象，然后回收没有标记的对象。MajorGC的耗时比较长，因为要扫描再回收。MajorGC会产生内存碎片，为了减少内存损耗，我们一般需要进行合并或者标记出来方便下次直接分配。	
当老年代也满了装不下的时候，就会抛出OOM（Out of Memory）异常。

### 永久代
指内存的永久保存区域，主要存放Class和Meta（元数据）的信息,Class在被加载的时候被放入永久区域. 它和和存放实例的区域不同,GC不会在主程序运行期对永久区域进行清理。所以这也导致了永久代的区域会随着加载的Class的增多而胀满，最终抛出OOM异常。
在Java8中，永久代已经被移除，被一个称为“元数据区”（元空间）的区域所取代。
元空间的本质和永久代类似，都是对JVM规范中方法区的实现。不过元空间与永久代之间最大的区别在于：元空间并不在虚拟机中，而是使用本地内存。因此，默认情况下，元空间的大小仅受本地内存限制。类的元数据放入 native memory, 字符串池和类的静态变量放入java堆中. 这样可以加载多少类的元数据就不再由MaxPermSize控制, 而由系统的实际可用空间来控制。
#### 采用元空间而不用永久代的几点原因：
* 为了解决永久代的OOM问题，元数据和class对象存在永久代中，容易出现性能问题和内存溢出
* 类及方法的信息等比较难确定其大小，因此对于永久代的大小指定比较困难，太小容易出现永久代溢出，太大则容易导致老年代溢出（因为堆空间有限，此消彼长）
* 永久代会为 GC 带来不必要的复杂度，并且回收效率偏低