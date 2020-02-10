---
title: YYYY引发的血案
date: 2020-02-09 23:40:48
tags: 线上问题
---

SimpleDateFormat时间转换戳里，MM和mm大小写分别代表不同的含义我们是知道的，没想到的是YYYY和yyyy也是有区别的。
<!--more-->线上出现了一个严重bug，体现是线上公司所有当天的审批数据都查询不到，当天是2019-12-31号。我们数据是根据月份切分存到es的，所以第一时间让运维查询201912这个分区的数据是否存在。运维发来的截图里惊奇的发现有个202012的这个分区。这可是需要再等一年时间才会来临的分区，所以第一时间就是我们把数据存错分区了。追踪代码发现我们存数据的时候根据创建时间转换成YYYYMM的字符串时间作为分区的，所以写了简单的一个测试用例如下：

```
public static void main(String[] args) {
        //模拟出即将跨年的最后一天的一个时间点
        String dateStr = "20191231 12:30:00";
        SimpleDateFormat df = new SimpleDateFormat("yyyyMMdd HH:mm:ss");
        Date date = null;
        try {
            date = df.parse(dateStr);
        } catch (ParseException e) {
            e.printStackTrace();
        }

        //重新获取年份
        SimpleDateFormat df2 = new SimpleDateFormat("YYYYMM");
        System.out.println(df2.format(date));
    }
```
上面代码运行结果，如果不认真看清楚的话，以为输出的是 “201912” ，但其实输出的结果是 “202012”。因为我们获取时间的时候用的是大写的“YYYY”时间戳。
那么“YYYY”和“yyyy”究竟有什么区别呢。
 YYYY是表示：当天所在的周属于的年份，一周从周日开始，周六结束，只要本周跨年，那么这周就算入下一年。
 yyyy：精确显示时间所属的年份。