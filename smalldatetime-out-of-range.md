---
title: smalldatetime out-of-range
date: 2020-02-13 11:32:02
tags: 线上问题
---

smalldatetime和datetime有区别，主要有时间范围和时间精度之分。
<!--more--> 先说精度之分，一般精确时间到毫秒的时候用datetime，而且绝大部分情况也是用datetime。但是如果有些字段只是精确到天，或者精确到分的话，则可以使用smalldatetime（时间格式看起来更简洁）。 再说时间范围的差异，datetime大致使用时间范围为1753-9999年，而smalldatetime的时间范围为1900-2079年。

先说一个线上问题吧，老公司迁移到新公司的时候一直失败，后来运维发来错误日志： The conversion of a varchar data type to a smalldatetime data type resulted in an out-of-range value。 （varchar数据转换成smalldatetime数据的时候发生数据超限）。理解这个意思后，查出新表里smalldatetime类型的字段time（仿名），然后申请查询老表里该字段最大值为2099-12-31。 然后在线下做个简单的更新测试

```
update TABLE1 set time = '2099-12-31' where id = 1 ;

[Err] 22007 - [SQL Server]The conversion of a varchar data type to a smalldatetime data type resulted in an out-of-range value.
01000 - [SQL Server]The statement has been terminated.
```
至此完全复现出了该问题，修复数据就跟着产品的要求来就行了。
以后再做迁移的时候还是要比较下两个字段类型是否完全一致，如果不是完全一致最好要搞清楚两者之间的微小差异。