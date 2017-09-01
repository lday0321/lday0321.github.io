---
title: Kafka Consumer Rebalance的演进
date: 2017-07-24 23:25:14
tags: 
	- Kafka
---

Kafka引入了Consumer Group的概念。在一个Consumer Group中的若干Consumer，会按照一定规则（均匀）分配同一topic下的所有partition，各自在相应partition上执行sub工作。当一个Group中有Consumer退出时，Group中的其他Consumer会对topic下的partition进行重新分配，并基于重新分配的结果继续执行sub工作。这个重新分配的过程被称为Rebalance。

对于Sub方的应用而言，Rebalance过程是透明的，Consumer Group Rebalance实现也经历了几个版本的演进，本文对Rebalance的实现方案进行大致的梳理和总结

<!-- more -->


# 方案一：基于Zookeeper的实现

在0.9.0.0版本之前(<=0.8.2.2)，Kafka的Consumer Rebalance方案是基于Zookeeper的Watcher来实现的。每个consumer group(cg)在zk下都维护一个"/consumers/[group_name]/ids"路径，在此路径下，使用临时节点记录属于此cg的消费者的Id，该Id信息由对应的consumer在启动时创建。还有两个与ids同一级别的节点，他们分别是：owners节点，记录了分区与消费者的对应关系（即：Topic A的Partition 0在Group G上实际由哪个Id的消费者负责消费）；offset节点，记录了此cg在某个分区上的消费位置（即：Group G对Topic A的Partition 0实际现在消费到了第几的offset）。每个Broker、Topic以及分区在zk上也都对应一个路径：

1. /brokers/ids/[broker_id]：记录了该broker的host、port信息，同时还记录了分配在此Broker上的Topic的分区列表。
2. /brokers/topics/[topic_name]：记录了每个partition的leader、isr等信息。
3. /brokers/topics/[topic_name]/partitions/[partition_num]/stat：记录了当前leader、选举epoch等信息

![](http://og43lpuu1.bkt.clouddn.com/kafka_consumer_rebalance_evolution/img/01_zk_brokers_path.png)

每个consumer都分别在"/consumers/[group_name]/ids"和"/brokers/ids"路径上注册一个Watcher。当"/consumers/[group_name]/ids"路径的子节点发生变化时，表示consumer group中的消费者出现了变化；当"/brokers/ids"路径的子节点发生变化时，表示Broker出现了增减。这样，通过Watcher，每个消费者就可以监听Consumer Group和Kafka集群的状态变化了。

![](http://og43lpuu1.bkt.clouddn.com/kafka_consumer_rebalance_evolution/img/02_zk_consumers_path.png)

这个方案看上去不错，但是验证依赖于zk集群，有两个比较严重的问题：

* 羊群效应(Herd Effect)：一个被Watch的zk节点变化，导致大量的watcher通知需要被发送给客户端，这会导致在通知期间其他操作的延迟。一般出现这种情况的主要原因就是没有找到客户端真正关注的点，也就是滥用Watcher的一种场景。例如，g1是Consumer Group A(G_A)中的一员，则他会去关注"/consumers/G_A/ids"路径的子节点变化，假设该Consumer Group A中还有一个成员g2，这里需要注意的是，Kafka允许同一cg中的两个成员，分别用于sub不同的topic，例如g1订阅的topic 1的消息，g2订阅的是topic 2上的消息，由于g1, g2都归属于G_A，则他们都会关注"/consumers/G_A/ids"，当g2退出时，路径"/consumers/G_A/ids"下关于g2的节点将消失，此时，g1将感知到节点变化，进而进入rebalance状态，然而g1，g2虽然同属于同一个group，但实际上他们是订阅着不同的topic，两者在工作上并无交集，因此，g2的退出，引起g1进行rebalance实际上是没有意义的，而在这个方案中，g1却无辜躺枪。

* 脑裂(Split Brain)：每个Consumer都是通过zk中保存的元数据来判断group中各其他成员的状态，以及broker的状态，进而分别进入各自的Rebalance，执行各自的Rebalance逻辑。由于zk是[只保证"最终一致性"，不保证在状态切换过程中个连接客户端看到的一致性("Simultaneously Consistent Cross-Client Views")](http://zookeeper.apache.org/doc/r3.4.6/zookeeperProgrammers.html#ch_zkGuarantees)，这带来的问题就是，不同的Consumer在同一时刻可能连接在不同的zk服务器上，看到的元数据就可能不一样，基于不一样的元数据，执行Rebalance就会产生不一致（冲突）的Rebalance结果，Rebalance的冲突，会到导致consumer的rebalance失败。


# 方案二：基于GroupCoordinator管理存储在zk上的Group元数据

由于上述的原因，Kafka社区对Rebalance方案进行了新的设计和讨论。方案二就是讨论产生的结果，其主要思想是：将全部的Consumer Group分成多个子集，每个Consumer Group子集在Broker端对应一个GroupCoordinator对其进行管理。通过Consumer向GroupCoordinator发起JoinGroup请求、或者Consumer的HeartBeat超时，促使GroupCoordinator感知Group成员变化，由GroupCoordinator在zk上添加Watcher，感知目标topic/partition的变化（在topic发生扩容时，会触发GroupCoordinator设置的Watcher）。基于上述变化，GroupCoordinator进行Rebalance操作。整个过程大致如下：

1. 当前消费者准备加入某Group或者GroupCoordinator发生故障转移时，消费者并不知道GroupCoordinator的网络位置，此时，消费者会向Kafka Broker集群中的任意一个Broker发送ConsuemrMetadataRequest请求，此请求中包含了其Consumer Group的GroupName，收到请求的Broker会返回ConsumerMetadataResponse响应，其中包含了管理此ConsumerGroup的GroupCoordinator的相关信息。

2. 消费者根据ConsumerMetadataResponse中的GroupCoordinator信息，连接到GroupCoordinator并周期性的发送HeartBeatRequest。发送心跳包的作用主要是为了告诉GroupCoordinator“我还活着”，GroupCoordinator会认为长时间未发送心跳包的消费者已经挂掉，进而触发GroupCoordinator的Rebalance逻辑。

3. 如果HeartbeatResponse中带有IllegalGeneration异常，说明GroupCoordinator发起了Rebalance操作，此时消费者进入Rebalance状态，发送JoinGroupRequest，等待GroupCoordinator完成Rebalance逻辑，下发新的partition信息。GroupCoordinator会根据收到的JoinGroupRequest信息，以及zk上的信息，完成rebalance逻辑，计算出各消费者应该负责哪些分区的消费。

4. GroupCoordinator计算出的Rebalance的结果将写入zk中保存，并且通过JoinGroupResponse，将结果下发给所有的消费者。消费者在收到JoinGroupResponse之后，则可以根据Response中的Rebalance安排，开始消费对应partition的数据。

5. 消费者成功称为Group成员之一（获取到了Rebalance结果）后，会周期性的发送HeartBeatRequest。如果HeartBeatResponse包含IllegalGeneration异常，则进入第3步。如果HeartBeatRequest发送失败，收到NotCoordinatorForGroup异常，则周期性的执行第1步，直至成功。

完整的方案二描述可参考[kafka wiki: Kafka 0.9 Consumer Rewrite Design](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+0.9+Consumer+Rewrite+Design)

其中consumer的状态机如下图所示：

![](http://og43lpuu1.bkt.clouddn.com/kafka_consumer_rebalance_evolution/img/03_solution2_consumer_state_diagram.jpg)


而新引入的GroupCoordinator的状态机如下图所示：

![](http://og43lpuu1.bkt.clouddn.com/kafka_consumer_rebalance_evolution/img/04_solution2_coordinator_state_diagram.jpg)

如上面的描述我们可以看到，方案二将Rebalance的逻辑交由GroupCoordinator来完成，所有消费者变成被动的结果接收方，这样一来，羊群效应和脑裂问题都得到了很好的解决。然后，这一方案却带来了新的问题：

* partition分配的逻辑完全是在GroupCoordinator上完成的，这需要整个分配策略在Broker中都定义好，并加载之。当要使用新的分配策略时，就需要修改服务端代码或者配置，之后重启服务，这显然比较麻烦

* partition的分配方式根据业务的不同会有很多灵活的需求，从逻辑上看，这种灵活的分配需求来自消费端，而并非broker端，当消费端有新的需求时，需要调整的却是Broker端，显然有些怪异，也不够灵活。




# 方案三：Kafka 0.9版本的最终实现方案

基于上述方案二的讨论，最终在Kafka 0.9版本中，Rebalance的重新设计将分区分配的工作放到了消费者这端来完成，而Consumer Group的管理工作则沿用了方案二的设计由GroupCoordinator来处理。这样的实现，让不同的模块了关注不同的逻辑，实现业务的切分和解耦。

方案三的协议是基于方案二的改进。原来JoinGroupRequest的处理逻辑被拆分成两个阶段，分别是Join Group阶段和Synchronize Group State阶段。当消费者找到管理当前Consumer Group的GroupCoordinator后，就会进入Join Group阶段，Consumer首先向GroupCoordinator发送JoinGroupRequest请求，其中包含了消费者的相关信息；GroupCoordinator在收到了消费者发来的JoinGroupRequest请求后，会暂存消费者的信息，待在一定时间内收集到全部消费者信息后，则将根据从所有消费者中选出1位消费者称为Group Leader，还会选取使用的分区分配策略，这些信息都将打包在JoinGroupResponse中返回给消费者

每个消费者都将受到JoinGroupResponse响应，但是只有Group Leader收到的JoinGroupResponse响应中包含了所有消费者信息。当消费者确定自己是Group Leader后，会根据消费者的信息以及选定的分区分配策略进行分区分配。

接下来进入Synchronize Group State阶段。每个消费者会发送SyncGroupRequest到GroupCoordinator，但是，只有Group Leader的SyncGroupRequest请求包含了分区的分配结果，GroupCoordinator根据Group Leader的分区分配结果，形成SyncGroupResponse，并返回给所有消费者。消费者在收到SyncGroupResponse后即可获知归属自己的分区信息

完整的Kafka 0.9版本的Rebalance方案可参见：[Kafka Client-side Assignment Proposal](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Client-side+Assignment+Proposal)

改进后的Consumer状态机如下图所示：

![](http://og43lpuu1.bkt.clouddn.com/kafka_consumer_rebalance_evolution/img/05_solution3_consumer_state_diagram.png)

改进后的GroupCoordinator状态机如下图所示：

![](http://og43lpuu1.bkt.clouddn.com/kafka_consumer_rebalance_evolution/img/06_solution3_coordinator_state_diagram.png)