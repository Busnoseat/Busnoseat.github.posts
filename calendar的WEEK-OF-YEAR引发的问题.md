---
title: calendar的WEEK_OF_YEAR引发的问题
date: 2020-05-18 16:28:46
tags: 线上问题
---

测试提出一个线上问题，大致是星期天的时候，首页上的的周指标没有数据。
<!--more-->奇怪的是周一至周六看的时候，首页上的周指标是没有问题的。作为开发的第一感觉，是和美式的周日算第一天这个设定相关。查到相关代码，果然周日算到了下一周。

```
public static void main(String[] args) throws ParseException {
        //2020-05-17是 2020年第20周的最后一天 星期天
        String dateStr = "2020-05-17";
        SimpleDateFormat df = new SimpleDateFormat("yyyy-MM-dd");
        Date date = df.parse(dateStr);

        Calendar cal = Calendar.getInstance();
        cal.setTime(date);
        //之前的代码就是没有设置setFirstDayOfWeek  WEEK_OF_YEAR的结果为21
        //并且DAY_OF_WEEK为1
        System.out.println(cal.get(Calendar.WEEK_OF_YEAR));
        System.out.println(cal.get(Calendar.DAY_OF_WEEK));

        //设置setFirstDayOfWeek为周一后  WEEK_OF_YEAR的结果正常显示为20 但是DAY_OF_WEEK还是为1
        cal.setFirstDayOfWeek(Calendar.MONDAY);
        System.out.println(cal.get(Calendar.WEEK_OF_YEAR));
        System.out.println(cal.get(Calendar.DAY_OF_WEEK));
    }
```

测试结果为:设置FirstDayOfWeek后，可以改变WEEK_OF_YEAR,但是并不能改变DAY_OF_WEEK，
外文解释:https://stackoverflow.com/questions/24554241/calendar-not-working-fine

```
When setting or getting the WEEK_OF_MONTH or WEEK_OF_YEAR fields,
Calendar must determine the first week of the month or year as a 
reference point. The first week of a month or year is defined as the
earliest seven day period beginning on getFirstDayOfWeek() and
containing at least getMinimalDaysInFirstWeek() days of that month or
year.  
（WEEK_OF_MONTH和WEEK_OF_YEAR方法，Calendar需要知道the first week，
The first week 又取决于FirstDayOfWeek开始的七天。
所以FirstDayOfWeek字段只是用来影响WEEK相关的操作，不影响day）  

```