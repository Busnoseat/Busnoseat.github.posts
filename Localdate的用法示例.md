---
title: Localdate的用法示例
date: 2020-06-08 22:02:05
tags: 优化
---

SimpleDateFormat不是线程安全的，在并发的时候会有取值错误的问题，calendar使用复杂。
<!--more-->所以在java8上添加了新的类，明确了日期时间概念，例如：瞬时（instant）、 长短（duration）、日期、时间、时区和周期。

## LocalDate
本地日期，不包含具体时间

* 当前时间
```
LocalDate localDate = LocalDate.now();
```
* 创建时间
```
LocalDate localDate = LocalDate.of(2020, 6, 8);
```
* 获取年月日周
```
localDate.getYear();
localDate.getMonth();
localDate.getMonthValue();
localDate.getDayOfMonth();
localDate.getDayOfWeek();
localDate.get(ChronoField.DAY_OF_WEEK)
localDate.getDayOfYear()
```
* 判断是否是同一天
```
 //LocalDate 重载了equal方法
   LocalDate localDate = LocalDate.now();
   LocalDate localDate1 = LocalDate.of(2020, 06, 8);
   System.out.println(localDate.equals(localDate1));
```
*	isLeapYear来判断是否是闰年
```
LocalDate localDate = LocalDate.now();
System.out.println(localDate.isLeapYear());
```


## LocalTime
本地时间，不包含具体日期
*	获取时
```
LocalTime localTime = LocalTime.now();
localTime.getHour()
localTime.get(ChronoField.HOUR_OF_DAY)

```

*	获取分
```
localTime.getMinute()
localTime.get(ChronoField.MINUTE_OF_HOUR)
```

*	获取秒
```
localTime.getSecond()
localTime.get(ChronoField.SECOND_OF_MINUTE)
```

## LocalDateTime
组合了日期和时间 但是不包含时差和时区信息
*	获取年月日 用法同LocalDate
```
localDateTime.getYear()
localDateTime.getMonthValue()
localDateTime.getDayOfMonth() 
```
*	获取时分秒 用法同LocalTime
```
localDateTime.getHour()
localDateTime.getMinute()
localDateTime.getSecond()
```
*	加日期
```
LocalDateTime localDateTime = LocalDateTime.now();
localDateTime = localDateTime.plusYears(1);
localDateTime = localDateTime.plusMonths(1);
localDateTime = localDateTime.plusDays(1);
System.out.println(localDateTime);
```
*	减日期
```
LocalDateTime localDateTime = LocalDateTime.now();
localDateTime = localDateTime.minusHours(1);
localDateTime = localDateTime.minusMinutes(1);
localDateTime = localDateTime.minusSeconds(1);
System.out.println(localDateTime);
```

## ZonedDateTime
最完整的日期时间，包含时区和相对UTC或格林威治的时差


## Period
用来比较时间相差值，注意是不跨年月日的比较
```
LocalDate localDate = LocalDate.of(2020, 6, 8);
LocalDate localDate2 = LocalDate.of(2021, 7, 9);
Period period = Period.between(localDate, localDate2);
// 结果为1 表示相差1年
period.getYears()
// 结果为1 表示相差1月 脑壳疼
period.getMonths()
// 结果为1 表示相差1天 真的是时间来比较的
period.getDays()
```


