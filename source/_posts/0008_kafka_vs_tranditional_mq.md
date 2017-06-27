---
title: Kafka与传统消息中间件的差异
date: 2017-06-27 22:21:21
tags: 
	- Kafka
---

因工作关系，之前稍微接触、了解过一些传统的消息中间件（RabbitMQ, ActiveMQ, ZeroMQ, Tibco EMS/FTL, IBM MQ/LLM以及我们自研的消息中间件）。最近的工作则一直是基于Kafka展开的。Kafka的很多设计和理念和传统的消息中间件不太一样，谈谈自己的浅薄认识。

<!-- more -->

# Kafka与传统消息中间件的差异特性

我认为Kafka与传统消息中间件的差异主要体现在一下几方面：

* 分布式的架构设计理念
* 容错性的考虑
* 同主题并行消费的思想
* 适合不同场景的可配置性
* 基于日志的消息存储
* 对慢消费者的处理方式


## 分布式的架构设计理念

Kafka诞生在“大数据”时代，显然，他具有很强的大数据烙印：分布式、水平可扩展。这是Kafka明显区别与传统消息中间件的地方。我接触的生产环境Kafka集群，往往会在十几台到几十台机器之间，集群规模相对于传统消息中间件而言，普遍偏大。传统的消息中间件，往往面对的是企业级应用，从整体架构来看，多数情况下，每个client都会与消息中间件的某个broker建立连接，所有的消息，都经由这个broker推送至client，同时client发送的消息也经由这个broker投递至目标节点。

最基本的传统消息中间件架构类似如下：

![](http://og43lpuu1.bkt.clouddn.com/kafka_vs_tranditional_mq/01_tranditional_mq_01.png)

当然，在这种基础架构之下，也会演化出更加鲁棒的架构来

![](http://og43lpuu1.bkt.clouddn.com/kafka_vs_tranditional_mq/02_tranditional_mq_02.png)

如上图所示，在原有broker的基础上，引入了热备的standby节点，当broker出现故障时，能够顺利接管，并继续提供服务。当然，传统消息中间件也在朝着分布式的方向发展，节点不再局限于一个，同样，也是有一个集群来为外界提供服务。

![](http://og43lpuu1.bkt.clouddn.com/kafka_vs_tranditional_mq/03_tranditional_mq_03.png)

但是，即使这样，我们也会看出，从client的角度出发，他始终是与其中一个broker建立连接，基于这个连接完成所有消息的接收与发送。

这与Kafka的架构设计是非常不同的，在Kafka集群中，topic是被“打散”分布在集群的不同节点上进行处理的，换言之，当用户需要pub或者sub不同topic时，client不再对着唯一的一个broker，他将根据topic的“位置”不同，向不同的broker发起连接，并执行pub,sub动作。

![](http://og43lpuu1.bkt.clouddn.com/kafka_vs_tranditional_mq/04_tranditional_mq_04.png)

这种打散的架构设计，使得Kafka集群的平行扩展变得更加容易。不同的客户端，针对不同的topic/partition，可以向不同的broker发起连接，这样整个系统的整体并发度就大大提高了。

## 容错性的考虑

和大多数传统的消息中间件不同，Kafka引入了replica的概念，对于一个topic上的每个partition，kafka会将数据根据配置复制到多个broker上，每个broker的这份数据，被称作该topic/partition的一个副本(replica)。当集群中该partition的leader broker挂掉后，会从replica slave中选择一个broker继续对外提供服务。基于replica机制，在topic层面，用户可以灵活选择容错级别，对于可靠性要求较高的topic，设置较大的replica，对于可靠性要求不那么高，而对延时，吞吐量要求更高的topic，则可以设置较小的replica。

![](http://og43lpuu1.bkt.clouddn.com/kafka_vs_tranditional_mq/05_kafka_replica.png)

目前，最新的Kafka版本，还引入了针对容错需求的分机架的replica分布策略，使得同一partition的多个replica，能够尽可能均匀的分布在不同的机架上，避免同一时间，由于硬件故障导致的同一机架上多server同时挂掉时，Kafka topic不可用问题。

## 同主题并行消费的思想

传统的消息中间件，同样提供pub/sub模式的消息发送，但是对于订阅方sub而言，一个主题只会有一个sub。如果pub速度过快，sub则会出现处理不及时的情况。在传统消息中间件上，要解决这个办法，则是主动的将消息pub到不同的topic，实际上是将这些不同的topic组成一个topic组，由pub方负责“均匀”的将消息推送到这一组topic组中，同时，在sub方则针对topic组中的每一个topic，启动sub端进行处理，通过这种方式来提供消息处理的并行度。这层处理逻辑，实际上是可以抽象的，抽象的结构，就是kafka中的topic/partition关系，以及消费端的consumer group的处理逻辑。Kafka中的topic/partition关系，就好比上面提到的传统消息中间件中的topic组。但在传统消息中间件上，pub端需手动的来分配将消息写入到哪个topic。Kafka中，这个分配过程被称作partitioning，用户只需提供Partitioner接口，用于确认消息应该推送到哪个partition，后续的处理逻辑，都将由kafka提供的producer库完成。而consumer group则好比上面提到的接收topic组中个topic数据的消费端。在一个consumer grouop中的消费端(sub端)，被看做地位均等的，各sub端会负责处理该topic中的一个或者多个partition。整个partition分配过程对于用户而言是透明的。用户只关注通过sub端接收消息，然后执行处理。Kafka在架构层面为sub端提供了自动调整逻辑。当同一个group中的consumer数发生变化，当该topic的partition数据发生变化时，整个consumer group上的rebalance便被触发，rebalance的过程，则是该group中的所有consumer重新分配partition的过程，整个过程对于上层应用而言完全透明。正式基于这种框架型的设计，使得Kafka的上层用户，可以不关心partition、rebalance的具体实现，他只需关注在业务消息的发送，以及业务消息的接收处理。

![](http://og43lpuu1.bkt.clouddn.com/kafka_vs_tranditional_mq/06_kafka_consumer_group.png)

由上图可以看出，基于传统消息中间件，也可以构建与Kafka topic/partition-consumer group类似的关系。但是，用户在使用Kafka时，只需提供简单的partitioner接口，只需简单的制定加入同一个consumer group，即可完成整个关系构建，而传统消息中间件，则需要自己手动去完成整个dispatcher，以及rebalance的过程。Kafka将这层关系的组织和调整，加入到了Kafka的整体架构设计及实现中，使得Kafka的使用方方便不少。

## 适合不同场景的可配置性

Kafka已经被验证可以用于不同体量、不同规模的系统中。之所以Kafka可以应用于各类场景，很重要的一点是：Kafka开放了众多的可配置项。对Procuder的配置，对Consumer的配置，以及更多的，针对Broker的配置。用户在使用Kafka时，可以针对自身的需求调整Kafka的配置项：

* 在最求高吞吐量的场景下，用户可以采用异步的发送模式，同时，在牺牲一定可靠性的基础上，进一步调整副本复制因子，进一步提升发送性能，调整partition数，提高sub端的并发度等等...
* 在对可靠性要求很高的场景，则正好反过来，使用完全的同步模式，ack=all设置，等等
* ...

结合自身需求，对Kafka进行参数调整，适应自身的业务需求。传统的消息中间件也提供一定的可配置性，但和Kafka相比，其可定制的程度，显然要低很多。


## 基于日志的消息存储

Kafka核心的数据存储是基于日志(commit log)的，这一点非常不同于传统的消息中间件。传统的消息中间件往往是基于内存的，当消息抵达目的地之后，内存中的消息便“消失”了。而Kafka却不同，消息在被broker接收后会被写入到以topic/partition为单元的log segment中，并持久化保留下来。后续的读取动作，将基于这些持久化的log segment来完成。基于log的处理方式，给kafka的功能扩展带来了很多便利，例如数据流的回放，Broker的快速恢复等。commit log并非Kafka独创，实际上，在数据库系统中，广泛使用commit log作为数据库备份的底层机制。


## 对慢消费者的处理方式

消息的传递一般分为两种模式：1). pull模式，由客户端主动拉取数据；2). push模式，由Broker端主动推送数据。pull模式的好处是作为实际消费者的客户端，可以根据自身的处理能力来选择何时获取消息，处理能力强，则以更大的速率拉取消息，处理能力弱，则低速拉取消息，这样，sub端不会因为pub端发送过快而被压垮。push模式的好处，则是消息处理更及时，因为不需要等待客户端主动发起请求，broker就会将到达的消息推送至客户端。传统的消息中间件往往是以push模式为主，我认为，这和传统消息中间件主要基于内存的消息存放方式有关。在传统消息中间的概念里，中间件只是消息的传送者，他不负责消息的存储，因此，当消息从一个点发往另一个点，消息中间件会以最快的方式转发过去，至于消息的存储，或者说持久化，不在传统消息中间件的优先考量范围内。而Kafka是基于日志的消息存储，他天生就能够对消息持久化，他也不担心慢消费者会占用大量的消息资源，因此Kafka选择的是pull模式，这种模式对慢消费者而言是非常友好的，慢消费者并不会 因为生产者速度很快而被打爆，压力被Kafka挡住。



