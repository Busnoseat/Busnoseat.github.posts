---
title: rabbitMq
date: 2022-12-13 23:10:00
tags: 中间件
---

rabbitMq
<!--more-->

![Image text](/asset/article/20221215/1.png)

## 组件概览
+ <font color="#0000dd">Broker</font>：一个RabbitMQ实例就是一个Broker
+ <font color="#0000dd">Virtual Host</font>：虚拟主机。相当于MySQL的DataBase ，一个Broker上可以存在多个vhost，vhost之间相互隔离。每个vhost都拥有自己的队列、交换机、绑定和权限机制。vhost必须在连接时指定，默认的vhost是/。
+ <font color="#0000dd">Exchange</font>：交换机，用来接收生产者发送的消息并将这些消息路由给服务器中的队列。
+ <font color="#0000dd">Queue</font>：消息队列，用来保存消息直到发送给消费者。它是消息的容器。一个消息可投入一个或多个队列。
+ <font color="#0000dd">Binding</font>：绑定关系，用于 消息队列和交换机之间的关联 。通过路由键（ Routing Key ）将交换机和消息队列关联起来
+ <font color="#0000dd">Channel</font>：管道，一条双向数据流通道。不管是发布消息、订阅队列还是接收消息，这些动作都是通过管道完成。因为对于操作系统来说，建立和销毁TCP都是非常昂贵的开销，所以引入了管道的概念，以复用一条TCP连接。
+ <font color="#0000dd">Connection</font>：生产者/消费者 与broker之间的TCP连接。
+ <font color="#0000dd">Publisher</font>：消息的生产者。
+ <font color="#0000dd">Consumer</font>：消息的消费者。
+ <font color="#0000dd">Message</font>：消息，它是由消息头和消息体组成。消息头则包括**Routing-Key** 、 Priority （优先级）等。


## 交换机类型
Exchange 分发消息给 Queue 时， Exchange 的类型对应不同的分发策略，有3种类型的 Exchange ：**Direct** 、 **Fanout**、 **Topic** 。
+ <font color="#0000dd">Direct </font>：消息中的 Routing Key 如果和 Binding 中的 Routing Key 完全一致， Exchange 就会将消息分发到对应的队列中。
+ <font color="#0000dd">Broker</font>：每个发到 Fanout 类型交换机的消息都会分发到所有绑定的队列上去。Fanout交换机没有 Routing Key 。它在三种类型的交换机中转发消息是最快的 。
+ <font color="#0000dd">Topic</font>：Topic交换机通过模式匹配分配消息，将 Routing Key 和某个模式进行匹配。它只能识别两个 通配符 ：#和*

## 消息生存时间
TTL（Time To Live）：生存时间。RabbitMQ支持消息的过期时间，一共2种。
+ <font color="#0000dd">在消息发送时进行指定</font> 
通过配置消息体的 Properties ，可以指定当前消息的过期时间。
+ <font color="#0000dd">在创建Exchange时指定</font> 
从进入消息队列开始计算，只要超过了队列的超时时间配置，那么消息会自动清除。

## confirm机制
生产者投递消息后，如果Broker收到消息，则会给生产者一个应答。生产者进行接受应答，用来确认这条消息是否正常的发送到了Broker，这种方式也是 消息的可靠性投递的核心保障！
![Image text](/asset/article/20221215/2.png)
### 如何实现confirm确认消息
1 <font color="#0000dd">在channel上开启确认模式</font>  
channel.confirmSelect()
2 <font color="#0000dd">在channel上开启监听</font>  
addConfirmListener，监听成功和失败的处理结果，根据具体的结果对消息进行重新发送或记录日志处理等后续操作

## return消息机制
我们的消息生产者，通过指定一个Exchange和Routing，把消息送达到某一个队列中去，然后我们的消费者监听队列进行消息的消费处理操作。
但是在某些情况下，如果我们在发送消息的时候，当前的exchange不存在或者指定的路由key路由不到，这个时候我们需要监听这种不可达消息，就需要使用到<font color="#0000dd">Returrn Listener</font>  
基础API中有个关键的配置项 Mandatory ：如果为true，监听器会收到路由不可达的消息，然后进行处理。如果为false，broker端会自动删除该消息。
同样，通过监听的方式， <font color="#0000dd">chennel.addReturnListener(ReturnListener rl) </font>传入已经重写过handleReturn方法的ReturnListener。

## 消费端ACK与NACK
消费端进行消费的时候，如果由于业务异常	可以进行日志的记录，然后进行补偿。但是对于服务器宕机等严重问题，我们需要<font color="#0000dd">手动ACK</font>保障消费端消费成功
![Image text](/asset/article/20221215/3.png)
如上代码，消息在 消费端重回队列 是为了对没有成功处理消息，把消息重新返回到Broker。一般来说，实际应用中都会关闭重回队列（ 避免进入死循环 ），也就是设置为false。

## 死信队列DLX
死信队列（DLX Dead-Letter-Exchange）：当消息在一个队列中变成死信之后，它会被重新推送到另一个队列，这个队列就是死信队列。
DLX也是一个正常的Exchange，和一般的Exchange没有区别，它能在任何的队列上被指定，实际上就是设置某个队列的属性。
当这个队列中有死信时，RabbitMQ就会自动的将这个消息重新发布到设置的Exchange上去，进而被路由到另一个队列。
