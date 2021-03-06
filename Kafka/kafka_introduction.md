# Kafka的介绍

## 简介

Apache Kafka是分布式发布－订阅消息系统。

Apache Kafka与传统消息系统相比，有一下不同：

* 它被设计为一个分布式系统，易于向外扩展；
* 它同时为发布和订阅提供高吞吐量；
* 它支持多订阅者，当失败时能自动平衡消费者；
* 它将消息持久化到磁盘，因此可用于批量消费，例如ETL，以及实时应用程序。

----
## 架构

首先，我介绍一下Kafka的基本概念。它的架构包括以下组件：

* 话题（Topic）是特定类型的消息流。消息是字节的有效负载（Payload），话题是消息的分类名或种子（Feed）名。
* 生产者（Producer）是能够发布消息到话题的任何对象。
* 已发布的消息保存在一组服务器中，它们被称为代理（Broker）或Kafka集群。
* 消费者可以订阅一个或多个话题，并从Broker拉数据，从而消费这些已发布的消息。

![Kafka生产者、消费者和代理环境]
(../charts/kafka_producer_consumer_broker_relation.png)

图1：Kafka生产者、消费者和代理环境

生产者可以选择自己喜欢的序列化方法对消息内容编码。为了提高效率，生产者可以在一个发布请求中发送一组消息。下面的代码演示了如何创建生产者并发送消息。

#### 生产者示例代码：
	producer = new Producer(…); 
	message = new Message(“test message str”.getBytes()); 
	set = new MessageSet(message); 
	producer.send(“topic1”, set); 
	
为了订阅话题，消费者首先为话题创建一个或多个消息流。发布到该话题的消息将被均衡地分发到这些流。每个消息流为不断产生的消息提供了迭代接口。然后消费者迭代流中的每一条消息，处理消息的有效负载。与传统迭代器不同，消息流迭代器永不停止。如果当前没有消息，迭代器将阻塞，直到有新的消息发布到该话题。Kafka同时支持点到点分发模型（Point-to-point delivery model），即多个消费者共同消费队列中某个消息的单个副本，以及发布-订阅模型（Publish-subscribe model），即多个消费者接收自己的消息副本。下面的代码演示了消费者如何使用消息。

#### 消费者示例代码：
	streams[] = Consumer.createMessageStreams(“topic1”, 1) 
	for (message : streams[0]) { 
	bytes = message.payload(); 
	// do something with the bytes 
	} 
	
Kafka的整体架构如图2所示。因为Kafka内在就是分布式的，一个Kafka集群通常包括多个代理。为了均衡负载，将话题分成多个分区，每个代理存储一或多个分区。多个生产者和消费者能够同时生产和获取消息。

![kafka架构]
(../charts/kafka_overall_architecture.png)

图2：Kafka架构

----

## Kafka存储

Kafka的存储布局非常简单。话题的每个分区对应一个逻辑日志。物理上，一个日志为相同大小的一组分段文件。每次生产者发布消息到一个分区，代理就将消息追加到最后一个段文件中。当发布的消息数量达到设定值或者经过一定的时间后，段文件真正写入磁盘中。写入完成后，消息公开给消费者。

***与传统的消息系统不同，Kafka系统中存储的消息没有明确的消息Id。***

消息通过日志中的逻辑偏移量来公开。这样就避免了维护配套密集寻址，用于映射消息ID到实际消息地址的随机存取索引结构的开销。消息ID是增量的，但不连续。要计算下一消息的ID，可以在其逻辑偏移的基础上加上当前消息的长度。

消费者始终从特定分区顺序地获取消息，如果消费者知道特定消息的偏移量，也就说明消费者已经消费了之前的所有消息。消费者向代理发出异步拉请求，准备字节缓冲区用于消费。每个异步拉请求都包含要消费的消息偏移量。Kafka利用sendfile API高效地从代理的日志段文件中分发字节给消费者。

![Kafka存储架构图]
(../charts/kafka_store_architecture.png)

图3：Kafka存储架构



