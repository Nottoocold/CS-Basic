不错的文字入门教程：https://developer.aliyun.com/article/769883

# RabbitMQ

RabbitMQ 是一个消息中间件：它接受并转发消息。你可以把它当做一个快递站点，当你要发送一个包

裹时，你把你的包裹放到快递站，快递员最终会把你的快递送到收件人那里，按照这种逻辑 RabbitMQ 是

一个快递站，一个快递员帮你传递快件。RabbitMQ 与快递站的主要区别在于，它不处理快件而是接收，

存储和转发消息数据。



## 功能：

- 流量消峰
- 应用解耦
- 异步处理



## 四大核心概念

- 生产者，产生数据发送消息的程序是生产者

- 交换机，是 RabbitMQ 非常重要的一个部件，一方面它接收来自生产者的消息，另一方面它将消息

  推送到队列中。交换机必须确切知道如何处理它接收到的消息，是将这些消息推送到特定队列还是推

  送到多个队列，亦或者是把消息丢弃，这个得有交换机类型决定

- 队列，是 RabbitMQ 内部使用的一种数据结构，尽管消息流经 RabbitMQ 和应用程序，但它们只能存

  储在队列中。队列仅受主机的内存和磁盘限制的约束，本质上是一个大的消息缓冲区。许多生产者可

  以将消息发送到一个队列，许多消费者可以尝试从一个队列接收数据。这就是我们使用队列的方式

- 消费者，消费与接收具有相似的含义。消费者大多时候是一个等待接收消息的程序。请注意生产者，消费

  者和消息中间件很多时候并不在同一机器上。同一个应用程序既可以是生产者又是可以是消费者。



## 工作原理

![](http://img.zqqiliyc.love/mq/202303020844893.png)

- Broker：接收和分发消息的应用，RabbitMQ Server 就是 Message Broker。

- Virtual host：出于多租户和安全因素设计的，把 AMQP 的基本组件划分到一个虚拟的分组中，类似

  于网络中的 namespace 概念。当多个不同的用户使用同一个 RabbitMQ server 提供的服务时，可以划分出多个 vhost，每个用户在自己的 vhost 创建 exchange／queue 等。

- Connection：producer／consumer 和 broker 之间的 TCP 连接。

- Channel：如果每一次访问 RabbitMQ 都建立一个 Connection，在消息量大的时候建立 TCP Connection 的开销将是巨大的，效率也较低。Channel 是在 connection 内部建立的逻辑连接，如果应用程序支持多线程，通常每个 thread 创建单独的 channel 进行通讯，AMQP method 包含了 channel id 帮助客户端和 message broker 识别 channel，所以 channel 之间是完全隔离的。**Channel 作为轻量级的**Connection **极大减少了操作系统建立** **TCP connection** 的开销。

- Exchange：message 到达 broker 的第一站，根据分发规则，匹配查询表中的 routing key，分发

  消息到 queue 中去。常用的类型有：direct (point-to-point), topic (publish-subscribe) and fanout 

  (multicast)。

- Queue：消息最终被送到这里等待被消费者取走。

- Binding：exchange 和 queue 之间的虚拟连接，binding 中可以包含 routing key，Binding 信息被保

  存到 exchange 中的查询表中，用于 message 的分发依据。

## 消息应答

**概念**：消费者完成一个任务可能需要一段时间，如果其中一个消费者处理一个长的任务并仅只完成

了部分突然它挂掉了，会发生什么情况。RabbitMQ 一旦向消费者传递了一条消息，便立即将该消

息标记为删除。在这种情况下，突然有个消费者挂掉了，我们将丢失正在处理的消息。以及后续

发送给该消费这的消息，因为它无法接收到。

为了保证消息在发送过程中不丢失，rabbitmq 引入消息应答机制，消息应答就是:**消费者在接**

**收到消息并且处理该消息之后，告诉 rabbitmq 它已经处理了，rabbitmq 可以把该消息删除了。**

 

- 自动应答，消息发送后立即被认为已经传送成功，这种模式需要在**高吞吐量和数据传输安全性方面做权**

**衡**,因为这种模式如果消息在接收到之前，消费者那边出现连接或者 channel 关闭，那么消息就丢

失了,当然另一方面这种模式消费者那边可以传递过载的消息，**没有对传递的消息数量进行限制**，

当然这样有可能使得消费者这边由于接收太多还来不及处理的消息，导致这些消息的积压，最终

使得内存耗尽，最终这些消费者线程被操作系统杀死，**所以这种模式仅适用在消费者可以高效并**

**以某种速率能够处理这些消息的情况下使用**。