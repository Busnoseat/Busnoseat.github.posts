---
title: sql中不走索引的场景
date: 2019-07-08 17:59:00
tags: sql
---

本文完全转载。原文地址： https://www.cnblogs.com/b3051/p/8478500.html 感谢作者。 
索引创建完成后，如果查询的姿势不对，会造成事倍工半的效果，以下情况尤其注意！！！
<!--more-->

# select *，可能会导致不走索引
比如，你查询的是SELECT * FROM T WHERE Y=XXX;假如你的T表上有一个包含Y值的组合索引，但是优化器会认为需要一行行的扫描会更有效，这个时候，优化器可能会选择TABLE ACCESS FULL，但是如果换成了SELECT Y FROM T WHERE Y = XXX，优化器会直接去索引中找到Y的值，因为从B树中就可以找到相应的值。

# 单键值的b树索引列上存在null值，导致COUNT(*)不能走索引
如果在B树索引中有一个空值，那么查询诸如SELECT COUNT(*) FROM T 的时候，因为HASHSET中不能存储空值的，所以优化器不会走索引，有两种方式可以让索引有效，一种是SELECT COUNT(*) FROM T WHERE XXX IS NOT NULL或者把这个列的属性改为not null (不能为空)。

# 索引列上有函数运算，导致不走索引
如果在T表上有一个索引Y，但是你的查询语句是这样子SELECT * FROM T WHERE FUN(Y) = XXX。这个时候索引也不会被用到，因为你要查询的列中所有的行都需要被计算一遍，因此，如果要让这种sql语句的效率提高的话，在这个表上建立一个基于函数的索引，比如CREATE INDEX IDX FUNT ON T(FUN(Y));这种方式，等于Oracle会建立一个存储所有函数计算结果的值，再进行查询的时候就不需要进行计算了，因为很多函数存在不同返回值，因此必须标明这个函数是有固定返回值的。

# 隐式转换导致不走索引
索引不适用于隐式转换的情况，比如你的SELECT * FROM T WHERE Y = 5 在Y上面有一个索引，但是Y列是VARCHAR2的，那么Oracle会将上面的5进行一个隐式的转换，SELECT * FROM T WHERE TO_NUMBER(Y) = 5,这个时候也是有可能用不到索引的。

# 表的数据库小或者需要选择大部分数据，不走索引
在Oracle的初始化参数中，有一个参数是一次读取的数据块的数目，比如你的表只有几个数据块大小，而且可以被Oracle一次性抓取，那么就没有使用索引的必要了，因为抓取索引还需要去根据rowid从数据块中获取相应的元素值，因此在表特别小的情况下，索引没有用到是情理当中的事情。

# ！=或者<>(不等于），可能导致不走索引，也可能走 INDEX FAST FULL SCAN
例如select id  from test where id<>100

# 建立组合索引，但查询谓词并未使用组合索引的第一列，此处有一个INDEX SKIP SCAN概念,

# like '%liu' 百分号在前

# not in ,not exist
可以尝试把not in 或者 not exsts改成左连接的方式（前提是有子查询，并且子查询有where条件）。