---
title: rocketMq
date: 2022-12-13 23:15:00
tags: 中间件
---

rocketMq
<!--more-->

![Image text](/asset/article/20221215/4.png)

## 组件概览
+ <font color="#0000dd">Broker</font>：消息中转角色，负责 <font color="#0000dd">存储消息</font>，转发消息。Broker 是具体提供业务的服务器，单个Broker节点与所有的NameServer节点保持长连接及心跳，并会定时将 Topic 信息注册到NameServer，顺带一提底层的通信和连接都是<font color="#0000dd">基于Netty实现 </font>的。Broker 负责消息存储，以Topic为纬度支持轻量级的队列，单机可以支撑上万队列规模，支持消息推拉模型。官网上有数据显示：具有 上亿级消息堆积能力 ，同时可 严格保证消息的有序性 。

+ <font color="#0000dd">Topic</font>：主题！它是消息的第一级类型。比如一个电商系统可以分为：交易消息、物流消息等，一条消息必须有一个 Topic 。Topic 与生产者和消费者的关系非常松散，一个 Topic 可以有0个、1个、多个生产者向其发送消息，一个生产者也可以同时向不同的 Topic 发送消息。一个 Topic 也可以被 0个、1个、多个消费者订阅

+ <font color="#0000dd">Tag </font>：标签！可以看作子主题，它是消息的第二级类型，用于为用户提供额外的灵活性。使用标签，同一业务模块不同目的的消息就可以用相同Topic而不同的 Tag 来标识。比如交易消息又可以分为：交易创建消息、交易完成消息等，一条消息可以没有 Tag 。标签有助于保持您的代码干净和连贯，并且还可以为 RocketMQ 提供的查询系统提供帮助

+ <font color="#0000dd">MessageQueue</font>：一个Topic下可以设置多个消息队列，发送消息时执行该消息的Topic，RocketMQ会轮询该Topic下的所有队列将消息发出去。消息的物理管理单位。一个Topic下可以有多个Queue，Queue的引入使得消息的存储可以分布式集群化，具有了水平扩展能力

+ <font color="#0000dd">NameServer</font>：类似Kafka中的ZooKeeper，但NameServer集群之间是 没有通信 的，相对ZK来说更加 轻量 。它主要负责对于源数据的管理，包括了对于 Topic 和路由信息的管理。每个Broker在启动的时候会到NameServer注册，Producer在发送消息前会根据Topic去NameServer 获取对应Broker的路由信息 ，Consumer也会定时获取 Topic 的路由信息。

+ <font color="#0000dd">Producer </font>：生产者，支持三种方式发送消息：<font color="#0000dd">同步、异步、单向 </font>
**单向发送**： 消息发出去后，可以继续发送下一条消息或执行业务代码，不等待服务器回应，且 没有回调函数
**异步发送**： 消息发出去后，可以继续发送下一条消息或执行业务代码，不等待服务器回应， 有回调函数
**同步发送**： 消息发出去后，等待服务器响应成功或失败，才能继续后面的操作。

+ <font color="#0000dd">Consumer </font>：消费者，支持<font color="#0000dd">集群消费 和 广播消费 </font>
**集群消费**：该模式下一个消费者集群共同消费一个主题的多个队列，一个队列只会被一个消费者消费，如果某个消费者挂掉，分组内其它消费者会接替挂掉的消费者继续消费。
**广播消费**：会发给消费者组中的每一个消费者进行消费。相当于 RabbitMQ 的发布订阅模式。

+ <font color="#0000dd">Group</font>：分组，一个组可以订阅多个Topic。分为ProducerGroup，ConsumerGroup，代表某一类的生产者和消费者，一般来说同一个服务可以作为Group，同一个Group一般来说发送和消费的消息都是一样的

+ <font color="#0000dd">Offset</font>：在RocketMQ中，所有消息队列都是持久化，长度无限的数据结构，所谓长度无限是指队列中的每个存储单元都是定长，访问其中的存储单元使用Offset来访问，Offset为Java Long类型，64位，理论上在 100年内不会溢出，所以认为是长度无限。也可以认为Message Queue是一个长度无限的数组， Offset 就是下标。

## 运转流程
1. NameServer 先启动
2. Broker 启动时向 NameServer 注册
3. 生产者在发送某个主题的消息之前先从 NamerServer 获取 Broker 服务器地址列表（有可能是集群），然后根据负载均衡算法从列表中选择一台Broker 进行消息发送。
4. NameServer 与每台 Broker 服务器保持长连接，并间隔 30S 检测 Broker 是否存活，如果检测到Broker 宕机（使用心跳机制， 如果检测超120S），则从路由注册表中将其移除。
5. 消费者在订阅某个主题的消息之前从 NamerServer 获取 Broker 服务器地址列表（有可能是集群），但是消费者选择从 Broker 中 订阅消息，订阅规则由 Broker 配置决定

## 延时消息
开源版的RocketMQ不支持任意时间精度，仅支持特定的level，例如定时5s，10s，1min等。其中，level=0级表示不延时，level=1表示1级延时，level=2表示2级延时，以此类推

## 顺序消费
消息有序指的是可以按照消息的发送顺序来消费（FIFO）。RocketMQ可以严格的保证消息有序，可以分为 分区有序 或者 全局有序 。

## 事务消息
rabbitmq提供类似XA分布式事务功能，通过消息队列MQ事务消息能达到分布式事务的最终一致。
### 事务消息发送及提交：
1. 发送half消息
2. 服务端响应消息写入结果
3. 根据发送结果执行本地事务（如果写入失败，此时half消息对业务不可见，本地逻辑不执行）
4. 根据本地事务状态执行Commit或Rollback（Commit操作生成消息索引，消息对消费者可见）。

### 事务消息的补偿流程：
1. 对没有Commit/Rollback的事务消息（pending状态的消息），从服务端发起一次“回查”
2. Producer收到回查消息，检查回查消息对应的本地事务的状态
3. 根据本地事务状态，重新Commit或RollBack其中，补偿阶段用于解决消息Commit或Rollback发生超时或者失败的情况

### 事务消息状态：
事务消息共有三种状态：<font color="#0000dd">提交状态、回滚状态、中间状态</font>：
1. TransactionStatus.CommitTransaction：提交事务，它允许消费者消费此消息
2. TransactionStatus.RollbackTransaction：回滚事务，它代表该消息将被删除，不允许被消费。
3. TransactionStatus.Unkonwn：中间状态，它代表需要检查消息队列来确定消息状态

## RocketMQ的高可用机制
RocketMQ是天生支持分布式的，可以配置主从以及水平扩展。
Master角色的Broker支持读和写，Slave角色的Broker仅支持读，也就是 Producer只能和Master角色的Broker连接写入消息；Consumer可以连接 Master角色的Broker，也可以连接Slave角色的Broker来读取消息。

### 消息消费的高可用性
在Consumer的配置文件中，并不需要设置是从Master读还是从Slave读，当Master不可用或者繁忙的时候，Consumer会被自动切换到从Slave读。有了自动切换Consumer这种机制，当一个Master角色的机器出现故障后，Consumer仍然可以从Slave读取消息，不影响Consumer程序。这就达到了消费端的高可用性

### 消息发送的高可用性
在创建Topic的时候，把Topic的多个Message Queue创建在多个Broker组上（相同Broker名称，不同 brokerId的机器组成一个Broker组），这样当一个Broker组的Master不可用后，其他组的Master仍然可用，Producer仍然可以发送消息。

## 主从复制

* <font color="#0000dd">同步复制</font>： 同步复制方式是等Master和Slave均写成功后才反馈给客户端写成功状态。如果Master出故障， Slave上有全部的备份数据，容易恢复同步复制会增大数据写入延迟，降低系统吞吐量。
* <font color="#0000dd">异步复制</font>
异步复制方式是只要Master写成功 即可反馈给客户端写成功状态。在异步复制方式下，系统拥有较低的延迟和较高的吞吐量，但是如果Master出了故障，有些数据因为没有被写 入Slave，有可能会丢失

## 负载均衡
### producer的负载均衡
Producer端，每个实例在发消息的时候，默认会<font color="#0000dd">轮询</font> 所有的Message Queue发送，以达到让消息平均落在不同的Queue上。而由于Queue可以散落在不同的Broker，所以消息就发送到不同的Broker下，如下图：
![Image text](/asset/article/20221215/6.png)

### consumer的负载均衡
如果Consumer实例的数量比Message Queue的总数量还多的话， 多出来的Consumer实例将无法分到Queue ，也就无法消费到消息，也就无法起到分摊负载的作用了。所以需要控制让Queue的总数量大于等于Consumer的数量


## 死信队列
当一条消息消费失败，RocketMQ就会自动进行消息重试。而如果消息超过最大重试次数，RocketMQ就会认为这个消息有问题。但是此时，RocketMQ不会立刻将这个有问题的消息丢弃，而会将其发送到这个消费者组对应的一种特殊队列：死信队列。死信队列的名称是 %DLQ%+ConsumGroup 。

1. 一个死信队列对应一个Group ID， 而不是对应单个消费者实例。
2. 如果一个Group ID未产生死信消息，消息队列RocketMQ不会为其创建相应的死信队列
3. 一个死信队列包含了对应Group ID产生的所有死信消息，不论该消息属于哪个Topic