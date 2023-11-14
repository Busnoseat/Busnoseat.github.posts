---
title: sqlserver执行计划
date: 2020-08-10 17:01:58
tags: 数据库
---
学习查看sqlserver的执行计划 可以帮助我们快速的定位到慢sql的原因。
<!--more-->
## 怎么看执行计划
图形化执行计划是从上到下从右到左看的。
* ** 要看有没有走索引 **
* ** 输入对象是什么索引 ** 
* ** 输出对象是哪些出参 ** 
* ** 查询结果行数为多少 ** 

### 连线
![image](/asset/article/20200810/1.jpg)
* 越粗表示扫描影响的行数愈多。
* Actual Number of Rows  ** 扫描中实际影响的的行数 ** 。
* Estimated Number of Rows ** 预估扫描影响的行数 **。
* Estimated row size 操作符生成的行的估计大小（字节）。
* Estimated Data Size 预估影响的数据的大小。

### 当前步骤执行信息
![image](/asset/article/20200810/2.jpg)
这个tips的信息告诉我们** 执行的对象是什么 **，** 输出的是哪些对象**，采用的操作是什么，查找的数据是什么，使用的索引是什么，排序与否，预估cpu、I/O、影响行数，实际行数等信息

## 索引分类
### Table Scan（表扫描）
![image](/asset/article/20200810/3.jpg)
当表中没有聚集索引，又没有合适索引的情况下，会出现这个操作。这个操作是很耗性能的，他的出现也意味着优化器要遍历整张表去查找你所需要的数据。

### Clustered Index Scan(聚集索引扫描)、Index Scan（非聚集索引扫描）
![image](/asset/article/20200810/4.jpg)
* 聚集索引扫描：聚集索引的数据体积实际是就是表本身，也就是说表有多少行多少列，聚集所有就有多少行多少列，那么聚集索引扫描就跟表扫描差不多，也要进行全表扫描，遍历所有表数据，查找出你想要的数据

* 非聚集索引扫描: 非聚集索引的体积是根据你的索引创建情况而定的，可以只包含你要查询的列。那么进行非聚集索引扫描，便是你非聚集中包含的列的所有行进行遍历，查找出你想要的数据。

### Key Lookup(键值查找)
![image](/asset/article/20200810/5.jpg)
首先需要说的是查找，查找与扫描在性能上完全不是一个级别的，扫描需要遍历整张表，而查找只需要通过键值直接提取数据，返回结果，性能要好。
当你查找的列没有完全被非聚集索引包含，就需要使用键值查找在聚集索引上查找非聚集索引不包含的列

### RID Lookoup（RID查找)
![image](/asset/article/20200810/6.jpg)
跟键值查找类似，只不过RID查找，是需要查找的列没有完全被非聚集索引包含，而剩余的列所在的表又不存在聚集索引，不能键值查找，只能根据行表示Rid来查询数据。

### Clustered Index Seek（聚集索引查找）、Index Seek（非聚集索引查找）
![image](/asset/article/20200810/7.jpg)
* 聚集索引查找：聚集索引包含整个表的数据，也就是在聚集索引的数据上根据键值取数据。

* 非聚集索引查找: 非聚集索引包含创建索引时所包含列的数据，在这些非聚集索引的数据上根据键值取数据。

### Hash Match
![image](/asset/article/20200810/8.jpg)
* Hashing：在数据库中根据每一行的数据内容，转换成唯一符号格式，存放到临时哈希表中，当需要原始数据时，可以给还原回来。类似加密解密技术，但是他能更有效的支持数据查询。
* Hash Table：通过hashing处理，把数据以key/value的形式存储在表格中，在数据库中他被放在tempdb中。
* Hash Match: 会把其中较小的数据集，通过Hashing运算放入HashTable中，然后一行一行的遍历较大的数据集与HashTable进行相应的匹配拉取数据

### Nested Loops
![image](/asset/article/20200810/9.png)
把两个不同列的数据集汇总到一张表中。提示信息中的Output List中有两个数据集，下面的数据集（inner set）会一一扫描与上面的数据集（out 
set），知道扫描完为止，这个操作才算是完成

### Merge Join
![image](/asset/article/20200810/10.png)
这种关联算法是对两个已经排过序的集合进行合并。如果两个聚合是无序的则将先给集合排序再进行一一合并，由于是排过序的集合，左右两个集合自上而下合并效率是相当快的

## 清除缓存的执行计划
```
dbcc freeprocache
dbcc flushprocindb（db_id）
```

## 优化建议
优化的目的是为了尽可能走index seek，杜绝index scan，完全禁止table scan。如果需要关联查询，尽可能的走Nested join,杜绝Hash Match和Merge join
* select * 通常情况下聚集索引会比非聚集索引更优
* 如果出现Nested Loops，需要查下是否需要聚集索引，非聚集索引是否可以包含所有需要的列
* Hash Match连接操作更适合于需要做Hashing算法集合很小的连接
* Merge Join时需要检查下原有的集合是否已经有排序，如果没有排序，使用索引能否解决
* 出现表扫描，聚集索引扫描，非聚集索引扫描时，考虑语句是否可以加where限制，select * 是否可以去除不必要的列
* 出现Rid查找时，是否可以加索引优化解决
* 在计划中看到不是你想要的索引时，看能否在语句中强制使用你想用的索引解决问题，强制使用索引的办法Select CluName1,CluName2 from Table with(index=IndexName)
* 看到不是你想要的连接算法时，尝试强制使用你想要的算法解决问题。强制使用连接算法的语句：select * from t1 left join t2 on t1.id=t2.id option(Hash/Loop/Merge Join)
* 看到不是你想要的聚合算法是，尝试强制使用你想要的聚合算法。强制使用聚合算法的语句示例：select  age ,count(age) as cnt from t1 group by age  option(order/hash group)
* 看到不是你想要的解析执行顺序是，或这解析顺序耗时过大时，尝试强制使用你定的执行顺序。option（force order）
* 看到有多个线程来合并执行你的sql语句而影响到性能时，尝试强制是不并行操作。option（maxdop 1）
* 不操作多余的列，多余的行，不做无必要的聚合，排序