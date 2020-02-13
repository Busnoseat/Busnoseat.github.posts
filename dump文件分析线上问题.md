---
title: dump文件分析线上问题
date: 2020-02-01 15:36:09
tags: 线上问题
---

一般运维都会配置dump文件生成规则,当线上服务发生宕机后，我们要做的就是第一时间就是像运维索要dunp文件并分析。
<!--more-->这里先简化一个完整的dump文件分析线上问题。

## 模拟获取dunp文件

在idea的VM options里配置dump生成路径

```
-Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=D:\soft
```
启动测试类
```
public class TestDump {

    class Student {
        Long[] a;

        public Student(Long[] a) {
            this.a = a;
        }
    }

    @Test
    public void test1() {
        List a=new ArrayList<>();
        for (int i = 0; i < 1000000; i++) {
            Student s = new Student(new Long[1024 * 1024]);
            a.add(s);
        }
    }
    
}
```
运行结果如下图：
![Image text](/asset/article/20200201/1.png)

## visualVm打卡dump文件
采用jdk自带的jvisualVm打开dump文件
![Image text](/asset/article/20200201/2.png)
装入生成的dump文件
![Image text](/asset/article/20200201/3.png)
在概要里可以大概看到发生宕机前处理的相关类
![Image text](/asset/article/20200201/4.png)
更准确的是切换到“类”标签下，按照内存大小排序，查看大小占比如下图
![Image text](/asset/article/20200201/5.png)
明显大量内存被分配到了long[]里，查询TestDump类long[]的地方分析就完事了。



感谢原文： https://blog.csdn.net/better_mouse/article/details/89449580 受益匪浅