---
title: mysql的存储引擎
date: 2020-05-27 17:45:55
tags: 数据库
---

InnoDB和MyISAM都是mysql的存储引擎，在MySQL 5.5版本前，默认的存储引擎为MyISAM。在那之后MySQL的默认存储引擎改为InnoDB。
<!--more-->

# InnoDB和MyISAM的区别
*	InnoDB 支持事务，MyISAM 不支持事务，这是 MySQL 将默认存储引擎从 MyISAM 变成 InnoDB 的重要原因之一。对于InnoDB每一条SQL语言都默认封装成事务，自动提交，这样会影响速度，所以最好把多条SQL语言放在begin和commit之间，组成一个事务
*	InnoDB 支持外键，而 MyISAM 不支持。对一个包含外键的 InnoDB 表转为 MYISAM 会失败
*	InnoDB是聚簇索引（叶子节点存数据），MyISAM是非聚簇索引（叶子节点存指针）。
	*	InnoDB是聚集索引，使用B+Tree作为索引结构，数据文件是和（主键）索引绑在一起的（表数据文件本身就是按B+Tree组织的一个索引结构），必须要有主键，通过主键索引效率很高。但是辅助索引需要两次查询，先查询到主键，然后再通过主键查询到数据。因此，主键不应该过大，因为主键太大，其他索引也都会很大。
	*	MyISAM是非聚集索引，也是使用B+Tree作为索引结构，索引和数据文件是分离的，索引保存的是数据文件的指针。主键索引和辅助索引是独立的。
![image](/asset/article/20200527/1.png)

*	InnoDB不保存表的具体行数，执行select count(*) from table时需要全表扫描。而MyISAM用一个变量保存了整个表的行数，执行上述语句时只需要读出该变量即可，速度很快（注意不能加有任何WHERE条件）

*	 InnoDB 最小的锁粒度是行锁，MyISAM 最小的锁粒度是表锁。一个更新语句会锁住整张表，导致其他查询和更新都会被阻塞，因此并发访问受限。这也是 MySQL 将默认存储引擎从 MyISAM 变成 InnoDB 的重要原因之一

# 使用场景
1	是否要支持事务，如果要请选择innodb，如果不需要可以考虑MyISAM
2	如果表中绝大多数都只是读查询，可以考虑MyISAM，如果既有读也有写，请使用InnoDB
3	系统奔溃后，MyISAM恢复起来更困难，能否接受
4	MySQL5.5版本开始Innodb已经成为Mysql的默认引擎(之前是MyISAM)，说明其优势是有目共睹的，如果你不知道用什么，那就用InnoDB，至少不会差

# InnoDB为什么推荐使用自增ID作为主键
自增ID可以保证每次插入时B+索引是从右边扩展的，可以避免B+树和频繁合并和分裂（对比使用UUID）。如果使用字符串主键和随机主键，会使得数据随机插入，效率比较差

# innodb引擎的4大特性
*	插入缓冲（insert buffer)
*	二次写(double write)
*	自适应哈希索引(ahi)
*	预读(read ahead)

【相关文献】 
https://www.zhihu.com/question/20596402
https://blog.csdn.net/qq_35642036/article/details/82820178