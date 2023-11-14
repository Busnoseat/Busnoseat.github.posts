---
title: mysql分析慢sql的命令
date: 2020-05-25 19:26:15
tags: 数据库
---
随着线上的数据增多，sql性能会逐渐体现出来，这个时候就体现出sql优化的重要性。
<!--more-->
## show status命令了解各种SQL的执行频率
show [session|global] status可以根据需要加上参数“session”或者“global”来显示session级（当前连接）的统计结果和global级（自数据库上次启动至今）的统计结果。如果不写，默认使用的参数是“session”。
```
mysql> show status like 'Com_%';
Com_select：执行SELECT操作的次数，一次查询只累加1。
Com_insert：执行INSERT操作的次数，对于批量插入的INSERT操作，只累加一次。
Com_update：执行UPDATE操作的次数。
Com_delete：执行DELETE操作的次数。
...
 
#针对InnoDB存储引擎
mysql> show status like 'Innodb_%';
Innodb_rows_read：SELECT查询返回的行数。
Innodb_rows_inserted：执行INSERT操作插入的行数。
Innodb_rows_updated：执行UPDATE操作更新的行数。
Innodb_rows_deleted：执行DELETE操作删除的行数。
```
此外，以下几个参数便于用户了解数据库的基本情况。
* Connections:试图连接MySQL服务器的次数
* Uptime：服务器工作时间
* Slow_queries：慢查询的次数
---

## 定位执行效率较低的SQL语句
以下两种方式定位执行效率较低的SQL语句
* 通过慢查询日志定位那些执行效率较低的SQL语句，用**--log-slow-queries[= file_name]**选项启动时，mysqld写一个包含所有执行时间超过long_query_time秒的SQL语句的日志文件
* 使用 **show processlist**命令查看当前MySQL在进行的线程，包括线程的状态、是否锁表等，可以实时地查看 SQL 的执行情况，同时对一些锁表操作进行优化
---

## 通过EXPLAIN分析低效SQL的执行计划
```
EXPLAIN select * from qf_marketing_realhouse.tp_property_sync where propertyNo='YXEB0100024';
 
id: 1
select_type: SIMPLE
table: tp_property_sync
type: ref   
possible_keys: propertyNo   
key: NULL
key_len: NULL
ref: const   
rows: 1
Extra: Using where
```
*	select_type: 表示SELECT的类型，常见的取值有SIMPLE（简单表，即不使用表连接或者子查询）、PRIMARY（主查询，即外层的查询）、UNION（UNION中的第二个或者后面的查询语句）、SUBQUERY（子查询中的第一个SELECT）等。
*	table：输出结果集的表。
*	type：表示MySQL在表中找到所需行的方式，或者叫访问类型，常见类型
	*	type=ALL，全表扫描，MySQL遍历全表来找到匹配的行
	*	type=index，索引全扫描，MySQL遍历整个索引来查询匹配的行
	*	type=range，索引范围扫描，常见于<、<=、>、>=、between等操作符
	*	type=ref，使用非唯一索引扫描或唯一索引的前缀扫描，返回匹配某个单独值的记录行
	*	type=eq_ref，类似ref，区别就在使用的索引是唯一索引，对于每个索引键值，表中只有一条记录匹配；简单来说，就是多表连接中使用 primary key或者 unique index作为关联条件
	*	type=const/system，单表中最多有一个匹配行，查询起来非常迅速，所以这个匹配行中的其他列的值可以被优化器在当前查询中当作常量来处理，例如，根据主键 primary key或者唯一索引 unique index进行的查询
	*	type=NULL，MySQL不用访问表或者索引，直接就能够得到结果
*	possible_keys：表示查询时可能使用的索引
*	key：表示实际使用的索引
*	key_len：使用到索引字段的长度
*	rows：扫描行的数量
*	Extra：执行情况的说明和描述，包含不适合在其他列中显示但是对执行计划非常重要的额外信息
tip:MySQL 4.1开始引入了 explain extended命令，通过 explain extended加上 show warnings，我们能够看到在SQL真正被执行之前优化器做了哪些SQL改写.
    MySQL 5.1开始支持分区功能，同时 explain命令也增加了对分区的支持。可以通过 explain partitions命令查看SQL所访问的分区。


## 通过show profile分析SQL
MySQL 从 5.0.37 版本开始增加了对 show profiles 和 show profile 语句的支持。通过have_profiling参数，能够看到当前MySQL是否支持profile：
```
mysql> select @@have_profiling;
+------------------+
| @@have_profiling |
+------------------+
| YES |
+------------------+
1 row in set (0.00 sec)
```
默认profiling是关闭的，可以通过set语句在Session级别开启profiling：
```
mysql> select @@profiling;
+-------------+
| @@profiling |
+-------------+
| 0 |
+-------------+
1 row in set (0.02 sec)
mysql> set profiling=1;
Query OK, 0 rows affected (0.00 sec)
```
通过profile，我们能够更清楚地了解SQL执行的过程.show profile能够在做SQL优化时帮助我们了解时间都耗费到哪里去了
tip:MyISAM表有表元数据的缓存（例如行数，即COUNT(星)值），那么对一个MyISAM表的COUNT(星)是不需要消耗太多资源的，而对于 InnoDB 来说，就没有这种元数据缓存，COUNT(星)执行得较慢。
通过 **show profiles**语句，可看到当前执行SQL的Query ID;
通过 **show profile for query**语句,能够看到执行过程中线程的每个状态和消耗的时间,在获取到最消耗时间的线程状态后，MySQL支持进一步选择 all、cpu、block io、context switch、page faults等明细类型来查看MySQL在使用什么资源上耗费了过高的时间.

## 通过 trace分析优化器如何选择执行计划
MySQL 5.6提供了对SQL的跟踪 trace，通过 trace文件能够进一步了解为什么优化器选择A执行计划而不选择B执行计划，帮助我们更好地理解优化器的行为。
使用方式：首先打开trace，设置格式为JSON，设置 trace最大能够使用的内存大小，避免解析过程中因为默认内存过小而不能够完整显示。
```
mysql> SET OPTIMIZER_TRACE="enabled=on",END_MARKERS_IN_JSON=on;
Query OK, 0 rows affected (0.03 sec)
mysql> SET OPTIMIZER_TRACE_MAX_MEM_SIZE=1000000;
Query OK, 0 rows affected (0.00 sec)
```
在执行想做trace的SQL语句后，检查INFORMATION_SCHEMA.OPTIMIZER_TRACE就可以知道MySQL是如何执行SQL的：
```
mysql> SELECT * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE\G
```
最后会输出一个json格式跟踪文件。