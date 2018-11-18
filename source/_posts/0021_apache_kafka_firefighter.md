---
title: Apache Kafka的分布式系统消防员(Controller Broker)
date: 2018-11-06 22:52:14
tags: 
	- kafka
---

作为分布式系统的Kafka，在管理、协调分布式节点，处理各类分布式系统事件时，都依赖于Controller Broker来完成。Controller Broker的工作包括：
1. Broker节点上、下线时，处理Broker节点的故障转移。
2. Topic新建或删除时，Partition扩容时，处理Partition的分配。
3. 管理所有Partition的状态机，以及所有Replica的状态机，处理状态机的变化。

以下内容介绍了Controller Broker的具体功能、逻辑，它翻译自：[Apache Kafka’s Distributed System Firefighter](https://hackernoon.com/apache-kafkas-distributed-system-firefighter-the-controller-broker-1afca1eae302)

<!-- more -->

# 简介

Kafka是一个在不断发展的分布式流式处理平台。他是构建可维护、可扩展的数据管道的首选解决方案。如果你还不太熟悉Kafka，确保你先阅读过我的另外一篇文章[《Kafka详细介绍》](https://hackernoon.com/thorough-introduction-to-apache-kafka-6fbf2989bbc1)

接上述文章，我认为，如果我们能更深入的了解Kafka的内部工作机理将会更有裨益。

今天我想向你介绍的是Controller，Kafka集群的重要节点。他用于保证整个分布式Kafka系统的健康和功能正确。

# Controller Broker

分布式系统是需要被协调的。当有事件发生时，系统中的节点需要有组织的行动。最终，需要某个人能够决定集群如何做出反应以及向Brokers发布一些指令。

这里的某个人，被叫做Controller。Controller也不是太复杂，他就是在一个普通的Broker之上，附加了一些额外的责任。也就是说，他仍然需要负责分区，仍然有读写请求经由他处理，也仍然负责备份数据。

他的额外责任中最重要的部分，就是跟踪集群中其他节点的状态，并且在有节点离开、加入、或者失效的时候，采取适当的措施。这些措施就包括重新平衡partition分布以及安排新的partition leader。

一个Kafka集群中，总是只有一个Controller。

## 职责

一个Controller Broker具有很多额外的职责。这些职责主要是一些管理类工作。其中就包括：创建/删除topic，增加partition（分配partition的leader）以及处理Brokers退出集群的场景。

### 处理节点离开集群

当一个节点因事故或者主动关闭而离开Kafka集群时，曾由该节点作为leader的partition，将变得不再可用（记住，client只从partition的leader读取或者写入数据）。因此，为了能尽量减少不可服务时间，尽快选出一个替代的leader就变得非常重要。

Controller就是那个会去响应其他broker失败事件的broker。他从ZK的watch上获得通知。ZK的一个watch是zk上针对某一数据的订阅。当这个数据发生变化时，zk将会通知那些订阅了该数据的所有人。zk的watch对于Kafka而言是至关重要的--他是Controller的输入。

这里，被跟踪的数据，就是集群的broker集合。

如下图所示，因为超过了zk会话出错时间(每一个Kafka节点都和Zk之间保持心跳，以确保他们和Zk之间的会话是存活的，一旦心跳停止，会话就过期)，Broker#2的id被从列表中删除。

Controller收到了这个通知，并且对该事件作出响应。他决定哪些节点将成为那些受影响partition的新的leader。他接着通知那些相关节点，他们应当成为新的leader，或者开始从新的leader处通过`LeaderAndIsr`请求来复制数据。

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0021_apache_kafka_firefighter/img/01_notify_leader_failure.png)

### 处理节点再次加入集群

合理的partition leader设置对于集群的负载均衡至关重要。如上图所示，当有节点故障退出后，一些节点接管了故障节点的工作，并成为比原来更多partition的leader。这些工作将会给各broker带来额外的负担，而这给集群的性能及整体运行状况带来了潜在的风险。

Kafka假设原来的leader分配设置是产生最佳平衡集群的最佳设置（当每个节点都存活时）。这些节点是所谓的**首选leaders** -- 指的是这些作为分区原始leader的Broker节点。由于Kafka还支持机架感知的leader选举算法（它试图将partition的leader和follower分配在不同的机架上，以提高对机架故障的容错能力），leader的放置位置与集群的可靠性紧密相关。

默认情况下(`auto.leader.rebalance.enabled=true`)，Kafka会去校验replica的当前leader是否为首选leader，如果首选节点是存活的，则会重新把他选为leader。

常见的Broker失败场景都是短暂失败场景，也就是说，Broker通常是过一段时间就会恢复。这就是为什么当一个节点退出集群时，这个节点元数据信息并不会被删除，同时，他作为follower的partition，也不会被重新分配给其他follower。

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0021_apache_kafka_firefighter/img/02_broker_failure.png)

当Controller发现有一个Broker加入到集群中时，他会通过该Broker的ID来确认，是否在该Broker已经存在partition了。如果有，Controller会将该状态、信息变化的事件通知新加入的以及原有的Broker。新的Broker将从当前leader那开始复制消息。

因为Controller是知道重新加入集群的broker原来的那些parition(曾经作为leader的partition)，为了达到集群尽可能平衡的目的，Controller会尝试将这些partition的leader角色重新分配给该Broker，

但是，请注意，重新加入的节点不能立即回收其过去的leader角色，因为，目前他还没有资格。

### 保持同步状态的副本

一个保持同步状态的副本(ISR)，是一个Broker，他完全追上了他跟踪的partition。换句话说，对于该partition而言，他不会落在最新消息后面。partition的leader他们自己有责任来维护哪些Broker是ISR，而哪些Broker不是。他们将这些信息存在ZooKeeper上。

在线时刻保持有足够的ISR是非常重要的。Kafka主要的可靠性和持久性保证依赖于数据复制。

对于将要从follower提升为leader的Broker而言，他首先必须是一个ISR。每一个partition都有一个ISR列表，这个列表由partition leader和Controller来维护更新。

从ISR中竞选出一个partition leader的过程被称作**干净leader选举**(clean leader选举)。如果有需要，用户也可以选择退出这种机制，在没有ISR存活，且leader又失败的特殊场景下，为了追求可用性而放弃了一致性，用户可以选择一个不在ISR中的副本Broker作为leader。

请记住，客户端仅仅会向leader生产消息，也仅仅会从leader消费消息。如果我们选择不具备最新数据的Broker作为leader，那么将会导致集群丢失消息！不仅仅是我们可能丢失消息，我们还有可能在consumer端造成消息冲突，因为被丢失的消息的offset，可能又会被新来的消息所使用。（*这样，就有可能造成，之前读到这个offset的消息是A，而non-ISR接管之后，再读该offset，读到的消息却可能是B，这是因为在non-ISR接管之后，该offset被写入了新的消息所致*）

不幸的是，即使是干净leader选举，这类冲突也还是有可能出现。（提示：不会出现相同offset的数据不一致，但是会比原来的消息多）一个ISR副本，也不是完全同步的。我的意思是，如果leader的最后一条消息的offset是100，一个ISR副本也许还没有拿到这条消息，副本的最后一条offset可能到达95或者99，亦或者80 -- 具体会达到哪个offset，取决于很多因素。因为，复制的过程是异步完成的，不可能确保follower正好拿到最后一条最新消息。

判定一个partition的follower是否与该分区的leader保持同步的条件如下：

* 他在最近的X秒内(通过`replica.lag.time.max.ms`配置)有通过partition的leader获取消息。仅仅是获取消息是不够的，获取消息的请求必须获取了达到leader日志最末尾offset的所有消息。这就确保了他尽可能的保持同步。

* 在最近的X秒内，他有向Zookeeper发起心跳（通过`zookeeper.session.timeout.ms`来配置）

### 不一致的持久性？

显然，存在这样一个可能的事件先后序列，leader挂掉，一个ISR副本作为替代，被选为leader，而我们丢失了一小部分消息。举例来说，如果一个leader在响应了一个follower的获取请求**之后**又保存了一些新的消息，这就有一个时间窗口，在这个窗口内，这些新的消息还没来得及被复制到follower。如果leader在这个时间窗口内挂掉，这些副本将任然被认为是同步的（也许，这个时间还未超过X秒），并被选举为新的leader。

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0021_apache_kafka_firefighter/img/03_isr_new_leader.png)

### Producer的acks设置

在上面的例子中，leader broker在将producer发来的消息保存到本地之后，便向producer返回了ack（producer设置了`acks=1`）。正好在他确认了新到的消息后，他便挂掉了。因为Broker 2是一个ISR副本。即使他此时缺失了offset为100的最新消息，Controller也将会选举他作为partition 1的新leader。

我们可以通过设置`acks=all`理论上来表明上述问题。`acks=all`的意思是，只有当所有ISR副本都已经成功复制了这条消息时，leader才能向producer确认该消息。不幸的事，这个设置将会降低集群的性能 -- 它将限制最大吞吐。partition的leader只有在他知道发来的消息被那些ISR follower都复制之后，他才能向producer确认消息写入成功。

因为副本使用拉模式，因此，leader只有在接收到该follower的第二次拉请求时，他才能确定消息已经被该副本保存。这也就导致producer需要在发起下一批消息发送之前等待更多的时间。

有些Kafka的使用场景，为了拥有更好的附加性能，他们会选择不采用上述配置(`acks=all`)

如果我们不想设置`acks=all`，会出现什么情况呢？我们会丢数据吗？某些consumers会不会读到那些丢失的消息？

长话短说--**不会**，不会发生这种事情。一些**已经生产**的消息也许会被丢失，但是这些消息绝对不会对consumer可见。这将确保端到端系统的一致性。


### 高水位Offset

Leader Broker不会将那些还未被所有ISR复制的消息返回回去。所有Broker自己都会跟踪一个所谓的**高水位offset** -- 就是所有ISR都完成复制的最新的offset。Kafka只会将小于等于高水位offset的消息返回给消费者，通过这种方式Kafka能够保证一致性，不可重复的读也不会发生。

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0021_apache_kafka_firefighter/img/04_high_watermark_offset.png)

### 脑裂

想象一下，Controller Broker挂掉了，Kafka集群必须迅速找到替代者，否则，当没有人来承担Controller角色时，集群健康状况将迅速恶化。

有一个问题是，你无法确切地知道一个Broker是完整的停止了，还是经历着一个间歇性的失败。无论如何，集群都必须继续运行，并且挑选一个新的Controller。当前，我们会发现我们有一个所谓的僵尸Controller。一个僵尸Controller是被集群认为已经挂掉了的原Controller节点，他又重新联机。另外一个Broker已经替代他的位置，被选为新的Controller，但是僵尸Controller自己也许并不知道。

这种事情很容易发生。举例来说，如果发生了令人讨厌的间歇性[网络分裂](https://aphyr.com/posts/288-the-network-is-reliable)或者Controller经历了一个足够长的[停止世界的GC暂停](http://blog.cloudera.com/blog/2011/02/avoiding-full-gcs-in-hbase-with-memstore-local-allocation-buffers-part-1/)--集群将会认为他已经挂掉了，然后挑选出一个新的Controller。在GC的场景下，从原来Controller的角度来看，一切都没有发生变化。Contorller Broker自己甚至都不知道他被暂停了，更不用说集群在没有他的情况下继续运行了。正因为如此，他会继续认为他就是当前的Controller，并执行相应的行为。这是分布式系统中很普遍的一种场景，这类场景被称为[脑裂](http://techthoughts.typepad.com/managing_computers/2007/10/split-brain-quo.html)。

我们来看一个例子。想象一下，当前Controller正好经历了一个长时间的停止世界的GC暂停。他的Zookeeper会话已经过期了，并且他注册的`/controller`节点已经被删除了。因为所有其他的Broker有在这个节点上设置Zookeeper观测通知，因此他的`/controller`节点被删除的事件已经通知到了所有其他Broker。

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0021_apache_kafka_firefighter/img/05_brain_split.png)

为了修复集群没有Controller的问题，每一个Broker都尝试让自己成为新的Controller。假设Broker 2首先创建了`/controller`节点而赢得了胜利，成为了新的Controller。

所有Broker都会收到这个节点被创建的通知，并且知道Broker 2是新的leader，除了Broker 3，他还在GC暂停中。很有可能这个通知事件由于这样或者那样的原因(例如，OS有太多的已被接收的连接正等待着被处理，而把这个通知事件给丢掉了)，最终没能抵达Broker 3。

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0021_apache_kafka_firefighter/img/06_new_leader_notification.png)

Broker 3的GC暂停最终将会结束，他将醒过来，并仍然认为他自己是Controller。记住，从他的角度来看，没有发生任何变化。

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0021_apache_kafka_firefighter/img/07_old_controller_wake_up.png)

你现在有两个Controller，而且他们会并行发出可能存在冲突的命令。显然你不希望这种事情在你的集群中发生，但如果不处理，可能导致严重的不一致。

如果Broker 2(新的Controller节点)收到了来自Broker 3的请求，他怎么知道Broker 3是不是最新的Controller？Broker 2知道的一切便是，对于同样的GC暂停可能在他身上也发生了！

这就需要有一个方法来辨别，谁才是真正的，当前集群的Controller。

有这样的方法！他是通过使用**纪元号**(也被称作隔离令牌)来完成的。一个纪元号是一个单调递增的数--如果老的leader的纪元号是1，新的leader的纪元号将会是2。通过很简单的相信拥有最大纪元号的Controller，Broker现在可以很容易的区分谁是真正的Controller。拥有最大纪元号的Controller必将是最新的Controller，因为纪元号最终是在递增。这个纪元号存在Zookeeper上（利用了他的[一致性保障](https://bowenli86.github.io/2016/07/04/distributed%20system/zookeeper/ZooKeeper-Consistency-Guarantees/)）

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0021_apache_kafka_firefighter/img/08_epoch_number.png)

现在，Broker 1记录了他看见的最新的`controllerEpoch`，这样一来，他将忽略带有较老纪元号的Controller的请求。

### 其他职责

Controller也会做一些其他更麻烦的事情。

* 创建新的Partition
* 创建新的Topic
* 删除Topic

以前，这些命令只能以一种糟糕的方式来完成--一个直接修改Zookeeper，并等待Controller对其修改做出响应的脚本。

自0.11版本和1.0版本开始，这些命令被调整为直接向Controller发请求。他们现在都能很方便的通过[AdminClient API](https://kafka.apache.org/documentation/#adminapi)被用户app调用，这些API会直接向Controller发送请求。

# 总结

在这篇简短的文章中，我们设法完整的解释了Kafka Controller是什么。我们看到，他是一个简单的Broker，仍然会作为partition的leader，并处理读/写请求，同时，他会具有一些额外的职责。

我们了解了Controller如何处理无响应的节点。首先，通过Zookeeper watch观测节点会话过期来确定节点无响应。接着，挑选一个新的Leader。最后，通过向其他Broker发送`LeaderAndIsr`请求来传播该消息。

我们也了解了Controller如何欢迎节点重回集群，以及他最终如何恢复集群的平衡。我们介绍了ISR的概念，并看到Kafka如何通过高水位offset来确保端到端一致性。

我们学到了Kafka通过使用纪元号来防止两个或更多节点认为自己是当前Controller的“脑裂”场景。我们一步一步解释了他是如何工作的。

Kafka是一个复杂的系统，由于其健康的社区，Kafka的功能和可靠性都在不断的增强。为了更好的了解他的发展，我建议加入[邮件列表](https://kafka.apache.org/contact)


