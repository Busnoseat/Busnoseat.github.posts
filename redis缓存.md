---
title: redis
date: 2019-09-25 16:03:56
tags: 缓存
---

 Redis 是完全开源免费的，遵守BSD协议，是一个高性能的key-value数据库。
 <!--more--> 

 # redis特点
 * 性能极高 – Redis能读的速度是110000次/s,写的速度是81000次/s 
 * 支持数据的持久化，可以将内存中的数据保存在磁盘中，重启的时候可以再次加载进行使用。
 * 丰富的数据类型 ，支持二进制案例的 Strings, Lists, Hashes, Sets 及 Ordered Sets 数据类型操作
 * 支持数据的备份，即master-slave模式的数据备份
 ---

 # 五大数据类型
 * string（字符串）
 1 string是redis最基本的类型，一个key对应一个value
 2 string类型是二进制安全的。意思是redis的string可以包含任何数据
 3 一个redis中字符串value最多可以是512M
 * list（列表）
 简单的字符串列表，按照插入顺序排序。你可以添加一个元素导列表的头部（左边）或者尾部（右边）。它的底层实际是个链表
 * set（集合）
 string类型的无序集合。它是通过HashTable实现的
 * hash（哈希）
  string 类型的 field 和 value 的映射表，hash 特别适合用于存储对象，类似Java里面的Map<String,Object>。
 * zset(sorted set：有序集合)
 1 zset 和 set 一样也是string类型元素的集合,且不允许重复的成员
 2 不同的是每个元素都会关联一个double类型的分数
 3 redis正是通过分数来为集合中的成员进行从小到大的排序。zset的成员是唯一的,但分数(score)却可以重复
 ---

 # 主从复制
 * 操作主库，数据也会同步到从库里。
 * 无法做master选举，主库宕机后需要手动将从库更改为主库。
 * slave节点配置文件里修改：  slaveof 主服务器的IP  端口号 。即可开启主从复制
 * 主从复制过程：  
 1 slave 连接到 master 后, 会向 master 发送 SYNC 命令.
 2 master接收到SYNC命令后，开启一个后台子进程执行BGSAVE,BGSAVE结束后生成.rdb文件，将rbd文件发给从库
 3 master开始执行BGSAVE时，会将该时间点后的新操作命令保存到缓存区，最终以redis命令协议的格式，将缓存区内容发给从库
 * 复制方式
 1 基于rdb文件复制
 2 无硬盘复制 配置rpli-diskless-sync yes
 3 增量复制 PSYNC master run id. offset
 ---

 # 哨兵模式
 * 配置文件为sentinel.conf
 * 配置节点: sentinel monitor mymaster 192,168,11,111 6379 2 最后的2为投票数
 * 客户端要访问的服务 IP 不是主节点，而是 sentiner 服务器的 IP
 * 监控：Sentinel 会不断地检查你的主服务器和从服务器是否运作正常
 * 自动故障迁移：主库不正常工作时，将主库其中的一个从库升级为主库，并将失效主库的其他从库改为复制新的主库。当客户端连接失效主库时，集群会返回客户端新主库地址
 * Sentinel 是一个分布式系统，可运行多个 Sentinel 进程来监控主库，并使用投票协议决定新主库
 * 是高可用方案, 但不是高性能方案（虽然主节点可以重新选举，但是主节点只有一个，大数据扛不住）
 ---

 # 集群
 * Redis 集群是一个提供在多个Redis间节点间共享数据的程序集
 * 不支持多个key的命令，因为不同节点间移动数据消耗性能
 * 自动分割数据到不同的节点。Reids有slot槽的概念: redis中有16384个. 根据key的 CRC16 算法, 取得的结果与槽数取模.落入的槽的索引是固定的. 然后根据节点数将槽的范围确定到每个节点上
 * 整个集群的部分节点失败或者不可达的情况下能够继续处理命令
 ---

# 持久化
## RDB (Redis DataBase)
按照一定策略定时同步内存的数据到磁盘,也就是行话讲的Snapshot快照，它恢复时是将快照文件直接读到内存里，生成的文件名为：文件名 dump.rdb
* 触发方式
1 配置的规则: save seconds exchange 。当在 seconds 指定的时间内, key 的数量更改大于 exchange 时发生快照.
2 使用命令save或者bgsave：save这个操作会暂时阻塞客户端请求. bgsave则不会阻塞。
3 flushall: 清除内存所有数据, 只要规则不为空, redis就会执行快照
* 快照原理
fork 复制一份当前进程的副本, 这个进程是子进程, 负责同步持久化到磁盘. 而父进程负责处理客户端请求.
* 缺点
1 在一定间隔时间做一次备份，所以如果redis意外宕掉的话，就会丢失最后一次快照后的所有修改；
2 fork的时候，内存中的数据被克隆了一份，大致2倍的膨胀性需要考虑。

## AOF (Append Only File)
以日志的形式来记录每个写操作，只许追加文件但不可以改写文件，它恢复时，将日志文件里的每个步骤从前到后执行一遍。AOF生成的文件名：appendonly.aof
* 启动方式
1 配置: appendonly yes 启动aof
2 配置 auto-aof-rewrite-percentage 100 当 aof 文件与上一次文件的大小相比, 超过配置的百分比就进行重写
3 配置 auto-aof-rewrite-min-size 64m 限制允许重写最小 aof 文件大小, 即小于64m时不重写
* rewrite原理
aof 重写是安全的,主进程fork出一条新进程来将内存中的数据库内容用命令的方式重写了一个新的aof文件
* 同步磁盘数据
aof机制会将命令记录到aof文件, 但实际是同步到操作系统的缓存区, 最终由操作系统同步到磁盘. 可以通过下面配置修改策略
1 appendsync always ： 每次执行写入就同步, 安全但影响性能
2 appendsync everysec ： 每一秒执行
3 appendsync no ： 不执行同步, 由操作系统去执行, 效率高但不安全
* 缺点
1 aof文件要远大于rdb文件，恢复速度慢于rdb
2 aof运行效率要慢于rdb,每秒同步策略效率较好，不同步效率和rdb相同

 ---

# redis事务
* Redis 事务可以一次执行多个命令，
一个事务从开始到执行会经历以下三个阶段：开始事务。命令入队。执行事务。
1 批量操作在发送 EXEC 命令前被放入队列缓存。
2 收到 EXEC 命令后进入事务执行，事务中任意命令执行失败，其余的命令依然被执行。
3 在事务执行过程，其他客户端提交的命令请求不会插入到事务执行命令序列中。

* Redis事务命令

|  命令   | 详解  |
|  ----   | ----  |
| MULTI   | 标记一个事务块的开始 |
| EXEC    | 执行所有事务块内的命令 |
| DISCARD | 取消事务，放弃执行事务块内的所有命令 |
| UNWATCH | 取消 WATCH 命令对所有 key 的监视 |

* 总结
单个 Redis 命令的执行是原子性的，但 Redis 没有在事务上增加任何维持原子性的机制，所以 Redis 事务的执行并不是原子性的。

 ---

# 发布订阅
* 原理
Redis的发布以及订阅有点类似于聊天，是一种消息通信模式。在这个模式中，发送者（发送信息的客户端）不是将信息直接发送给特定的接收者（接收信息的客户端）， 而是将信息发送给频道（channel）， 然后由频道将信息转发给所有对这个频道感兴趣的订阅者。
1 三个客户端订阅一个频道
2 当有新消息通过 PUBLISH 命令发送给频道 channel1 时， 这个消息就会被发送给订阅它的三个客户端
* 用途
分布式缓存中，多个内存缓存的clean，除了使用zookeeper之外，也可以使用redis的订阅发布功能来通知。胜在简易。

 ---

 # 

 【参考文献】：
 https://blog.csdn.net/qq_40804005/article/details/82919154
 https://www.runoob.com/redis/redis-intro.html
 https://www.cnblogs.com/hjwublog/p/5660578.html
 https://www.cnblogs.com/jimisun/p/10045772.html
 https://blog.csdn.net/Stream_who/article/details/86354391
 https://www.cnblogs.com/demingblog/p/10295236.html
 https://www.cnblogs.com/cheyunhua/p/10771694.html
 https://www.cnblogs.com/walkinhalo/p/10719688.html