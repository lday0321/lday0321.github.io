---
title: Kafka数据丢失及最新改进策略
date: 2017-10-08 22:46:14
tags: 
	- Kafka
---

上周在测试环境，无意间连续把两台kafka的磁盘打爆，导致broker相继挂掉。当我清理完磁盘，重启两台broker后，发现很有意思的现象：**kafka数据丢失！**从我们的处理日志上， 我们观察到有向一个topic的两个partition，分别写入了3条和7条，共计10条消息。但是，当我们恢复两台broker后，通过命令查看，发现这时该topic两个partition消息数量竟然都是0，而从zk上consumer group记录的信息来看，有cg的确已经消费过该topic的数据，且cg在对应partition上的offset分别为3,7。

<!-- more -->

这等怪事被遇上，何等幸运。怀着探秘的心里，搜寻了很多资料，终于在Kafka的wiki上找到了这篇文档说明，[KIP-101 - Alter Replication Protocol to use Leader Epoch rather than High Watermark for Truncation](https://cwiki.apache.org/confluence/display/KAFKA/KIP-101+-+Alter+Replication+Protocol+to+use+Leader+Epoch+rather+than+High+Watermark+for+Truncation)，这篇文档详细说明了0.11之前kafka的备份策略可能存在的问题，以及0.11版本对该类问题的设计重构及改进。以下是本人对该篇文档的理解性翻译，以及穿插了一些个人说明

## 使用Leader Epoch调整备份协议，不再使用High Watermark进行日志裁剪

### 术语定义

* Leader Epoch: 一个32位单调递增的数字，用于表示一个单一分区的leader世代，在所有消息中都包含该信息
* Leader Epoch Start Offset: 在一个leader epoch周期上当前leader标记的第一个offset
* Leader Epoch Sequence File: 一个记录leadership变更的文件，内容是一个leader epoch和leader epoch start offset的映射记录
* Leader Epoch Request: 由follower向leader发起的请求，用于获取相应世代的leader的Leader Epoch Start Offset信息。实际上，request请求参数是第n世代，则对应了response会返回第n+1世代的开始偏移位置。如果请求的n就是最新世代，也就意味着没有后续的n+1世代，此时，这个request返回的，就是当前世代leader的LEO(Log End Offset)。follower使用获取到的Start Offset来裁剪他自己的log

### 动机

有一些已知的情况，kafka可能在复制时向log中写入一些预期外的数据，或者更糟糕的，由于replica的偏差，导致无法对外提供服务。这个KIP提出对复制协议进行调整，确保这类问题不再发生。首先，我们来描述两种在现有复制协议下可能出现数据丢失场景，甚至是数据不一致的场景：

#### 场景1：先由HW引起日志裁剪，接着发生Leader选举

Kafka的复制协议分为两个阶段：第一个节阶段，follower获取消息，例如它先获取了消息m2。第二阶段，在下一轮RPC中（获取消息的RPC），他会向leader确认他收到了m2（*向leader确认收到m2的方法，就是在下一轮fetchMessage时，要求fetch的offset变成m2+1*）。我们假设，在第一阶段leader收到follower fetchMessage请求时，其他follower已经向leader发起并完成了fetch m2的请求。此时，leader会推进他自己的HW(High Watermark)，将自己的HW设置到m2。这个HW=m2的信息，会被加在后续所有followers的fetch Message的响应中，**（本轮这个follower的这次RPC响应中，leader HW还是设置成m1返回），具体可参考补充说明中的描述**，返回给follower。从这个过程我们可以看出：leader通过收集所有follower的确认，进而调整他自身的HW，而leader上的，更新后的HW，会在follower的后续RPC请求时，推送到其他follower

复制协议，同样还包含了一个过程：在follower的启动阶段，他会先根据他自身记录在自己High Watermark文件（replication-offset-checkpoint）中的信息，对他的数据log进行裁剪。接着，从leader处获取消息。这里就出现了一个问题：如果这个follower正在从leader中获取足够数据，并在追上当前leader之前，就由于当前leader挂掉，而被选举为新的leader，那么就会因为这个follower在启动时的这个裁剪动作，而导致有部分数据丢失（**丢失的数据，就是这个follower还没有追上当前leader的那部分数据**）

让我们来看一个示例。假设我们有两个broker: A & B。如下图所示：B在初始阶段是leader。(A)从leader(B)获取消息m2。此时follower(A)也已经有了m2，但是还没有进行下一轮fetchMessage的RPC。（*[注]：~~我个人认为在只有Broker A和B的情况下，不存在follower(A)已经拿到m2，但是HW不更新的情况，当有Broker C存在，且follower A先拿m2，这个时候follower A才有可能既拿到m2，又没有更新HW到m2，最后一个拿到m2的follower会在当次response时，就直接更新HW到m2了~~，实际上，即使是只有两个Broker， follower(A)在fetch m2时，获得的响应里面leader HW还是等于m1，因为leader (B)在处理fetch reqeust请求时，是首先记录更新前的HW(m1)，再接着计算是否需要将HW更新到m2，最后将更新前的HW通过fetch response返回给follower (A)*）因此，此时他还没有收到来自leader(B)的对m2进入HW的确认消息。而正好在这一个时间点上，follower(A)重启，根据(A)自身的HW文件，他执行日志的裁剪动作，由于在(A)重启前，(A)并没有收到leader对m2的确认，因此(A)的HW文件中，记录的仍然是该日志文件的HW是m1，因此裁剪时，会将m2从(A)的日志文件中抹掉。紧接着，在(A)完成裁剪之后，会向leader(B)发起fetchMessage的请求，而正好在这个时候，(B)挂掉了，(A)成为新的leader。此时，m2就这么永久的丢失了。（无论(B)之后是否恢复回来）

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0014_kafka_data_loss_and_new_mechanism/img/01_case_01.png)

这个问题的本质是应为follower需要额外的一轮RPC来更新他的HW。在这个额外一轮RPC时间间隔内，一次leader改变是可能的，进而就有可能由于follower裁剪了本地数据，并成为leader而导致数据丢失。对于这一问题，是有一些简单的解决方案的。其中之一就是，leader在等待所有follower确认follower自身的HW先变更到m2之后，leader再变更自己的HW到m2。（也就是说，让follower先增加HW，leader收集到所有follower都变更到HW之后，再变）。这其实也不够完美，因为他整个复制过程增加了新的一轮RPC。另一个解决方案，则是follower启动时，先不执行裁剪动作，先向leader发起fetch Message的请求，等响应回来，确认具体的HW，再执行裁剪。针对场景1，这个方案是可行的。但是他却不能解决我们下面要描述的第二个问题。

#### 场景2：在多副本异常失败之后，副本数据出现不一致

再次假设我们有两个broker，但是，这次我们出现了断点，此时两个broker都受到了影响（都被关了）。这种场景我们丢失数据是可以接受的，因为对于kafka而言，kafka只保证在最多n-1个失败的情况下，数据还存在。不幸的是，在这种情况下，可能出现不同主机上的日志数据不一致，甚至更糟的是，导致这个副本被卡住，不可用。

这其中的本质原因是因为消息是被异步刷到磁盘的。这意味着，在崩溃之后，各机器磁盘上可能写入了不等数量的消息。当他们恢复回来，其中任何一个都可能成为leader。如果leader正好是那个写入数据最少的主机，那么我们就会丢失数据。这种情况下丢失数据，实际上是可以接受的，属于系统已知限制。但问题是，这种情况下还有可能因为各主机上的副本数据不同，而使得副本整体数据不一致。

而由于我们还支持压缩消息集，在最坏的情况下，这会导致副本整个不可用。这种情况发生在，其中一个副本上压缩消息集的offset指向另一个副本上压缩消息集中的某一位置时。


![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0014_kafka_data_loss_and_new_mechanism/img/02_case_02.png)


### 解决方案

我们能通过引入Leader Epoch的概念来一并解决以上问题。他代表了一段leadership的标识，这个标识被加入到leader发出的每一个消息中。每一个副本都保存一个[LeaderEpoch==>StartOffset]的列表，用于标记在发生leader切换时，leader上的log状态。这个列表代替了在follower重启时裁剪数据时使用HW的功能（同时，对于每一个副本，都会保存到一个文件中）。所以，follower启动时，先向leader发送请求，获取LeaderEpoch列表，根据这个列表中的最后一项[epoch=1, offset=1]（也就是最近，最后一个leader，启动时写入第一个消息的地方），来裁剪本地的数据（将本地大于等于offset的数据裁减掉），而不是使用本地记录的HW信息来裁剪数据。通过这种方式，leader有效的告知了所有follower应该裁剪掉哪些数据。

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0014_kafka_data_loss_and_new_mechanism/img/04_leader_epoch.png)

我们可以以场景1为例，来看看整个实现过程：

在这个方案中，follower通过发送一个请求来确定自身和leader在日志数据上是否出现了不一致。他会向leadre发送一个LeaderEpochRequest请求，获取follower他自己所在世代的LeaderEpoch信息。在这个场景下，leader返回他的LEO(log end offset)（因为，此时follower向leader请求的参数是第0世代，按要求leader应该要回复第0+1世代的起始位置，但目前leader也还在第0世代，所以此时是回复第0世代的LEO信息）。当然，follower也有可能和当前leader隔了好几世代，这种情况下当前leader会返回follower请求的下一代(follower leader epoch + 1)的startOffset。也就是说LeaderEpoch响应包含的是请求的世代结束位置。

<font color="#0000FF">

**我是这么理解的**：在每一个replica上，有一个内存数据结构，这个结构，记录的是从有这个topic/partition创建以来，每一次leader世代的起始offset信息
```
[0: 0]
[1: 120]
[2: 240]
```
当然，每个replica也通过这个数据结构，知道，自己目前处于哪个世代，例如我的数据结构里面有[0,...], [1,...], [2,...]，那显然我们就处于第2世代。

同时，这个世代起始信息，也会写入到一个文件：Leader Epoch Sequence File，帮助replica重启恢复的时候，确认自己要从哪个世代恢复。假设，有一个replica，写入文件的世代信息是：
```
[0: 0]
[1: 120]
```
那他启动的时候，就知道，他挂掉之前，是处于第1世代和第2世代之间，因此，他会向leader去问，第1世代之后的世代，也就是第2世代的epoch start offset（其实变相的问，也就是问，我自己是在第1世代的， 我想知道的是第1世代的结束位置是在哪，对应的就是第1+1=2世代的起始位置在哪），例如ESO_2信息，拿到这个信息，再跟自己log的offset比，超过ESO_2的都应该被裁减掉。

同时，我认为follower在catch up的过程中，还应该把世代信息补齐，比如，这里他少了[2:240]，那么，follower在恢复的过程中，就需要一步步，把自己和leader之间缺少的世代信息都补齐，并且逐步刷新到自己的`Leader Epoch Sequence File`文件中。因为，他之后也有可能成为leader，而成为leader，就需要回到任何follower，关于第n世代的startOffset的问题。

</font>

在这个案例中，LeaderEpochResponse返回了offset 2。注意，这个返回值，和leader的HW是不一样的，leader此时的HW是offset 0。正因为返回了offset 2，因此，replica A在重启的时候，不会将offset 1位置上的m2裁减掉，m2没有丢失


![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0014_kafka_data_loss_and_new_mechanism/img/05_solution_01.png)

注意图上Request返回的是1，而不是文字描述的2， 我感觉是前期设计时的思路:返回n，和最终定稿时的决定:返回n+1，有一点小小的差异（从图上我们也看出，图的reqest还是叫LeaderGenerationRequest，而最终是叫：OffsetForLeaderEpochRequest），不过思路是一样的，只是判断的时候是用>还是用>=的差别。从图上我们也可以看出，当Replica B宕机，Replica A接管的时候，世代就由原来的0，升级到了1，LG=1, 后续的消息，例如m3，就是世代1（LG1）的消息了。

在来看看这个方案如何解决场景2：

当两个broker在挂掉之后恢复，broker B成为leader。他会接受m3，但是会带一个新的世代标志：LG1，也就是说这个时候已经演进到世代1了。现在broker A重启，变成一个follower。他会向leader发送一个LeaderEpoch request。此时，他请求时的参数是世代0，leader B会根据这个参数，返回世代1的起始位置：1（m3所在位置）。当follower A收到response时，他会知道，他本地位于offset 1，且自身出于第0世代的消息m2是一个孤悬的消息（也就是脏消息），此时follower A会将起裁剪，进而确保数据一致性。

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0014_kafka_data_loss_and_new_mechanism/img/06_solution_02.png)

注：这里我们需要注意，该方案无法保证在设置了`unclean.leader.election.enable=true`的情况下，不会出现数据不一致。


### 在Leader Epoch方案下，同时设置允许Unclean Leader Election，仍有可能数据不一致

在设置min.isr=1且unclean.leader.election.enabled=true的情况下，及时启用leader epochs方案，也还是有可能出现数据崩溃。 假设有两个broker (A, B), 1个topic，1个partition，2个replica， min.isr=1

场景：
1. 世代0， 向Broker A写入1条消息(offset A:0). 停止A, 再把B拉起来，成为leader
2. 世代1： 向Broker B写入1条消息(offset B:0), 停止B, 在把A拉起来，成为leader
3. 世代2： 向Broker A写入1条消息(offset A:1), 停止A, 在把B拉起来，成为leader
4. 世代3： 向Broker B写入1条消息(offset B:1)
5. 把A拉起来。他会向B发送一个请求世代2的EpochRequest，而B只有世代1,3的信息，没有2的，因此他回复世代3的offset(这个offset是1)，这时候A会根据1去裁剪日志，保留0，裁剪1。而实际上保留的0，也是不一致的数据。

这个现象的根本原因是，虽然B能告诉A，A的数据有问题，但是却不能精确的告知，不一致的起始位置。

一个方案是，通过完整比较broker之间的世代信息（我觉得，就是每个broker上都需要保持从创世纪开始的所有世代信息），检测是在什么时候造成了不一致，进而裁剪到0，或者裁剪到不一致的分叉点。然后重新拉取。然后，压缩的主题是的这两个选项也变得无法实现，因为任意的世代以及偏移量信息都有可能从日志中丢失。该信息可以在LeaderEpoch文件中保留和管理，但整个解决方案变得相当复杂。因此，放弃对unclean leader election场景的这种保证似乎是明智的，或者至少把它推到一个后续的KIP中去实现。


### 我所遇到的问题还原

如前所述，之所以会去了解Kafka的数据备份及恢复机制，是因为在测试环境中的Kafka broker接连宕机，导致数据丢失。在理解了kafka的数据备份及恢复机制后，我们再来看看，我在测试环境中遇到的问题，是如何产生的。

#### 现象

我们在测试环境公有两台Kafka Broker: 152, 153

按时间顺序，我们来回放一下整个过程：
1. 9/14， 2台broker正常运行
2. 9/14，创建topic：271.feeds_content_process_video.v1，2个partition，2个replica
3. 9/15 18:22:27,997，<font color="#FF0000">152由于磁盘被写满，挂掉，153独自运行</font>
4. 9/18 11:22:31左右，分别向partition-0，partition-1<font color="#0000FF">写入3,7共10条数据</font>
5. 9/18 12:04:11左右，通过271.feeds_content_process_video_cg消费该10条消息
6. 9/18 13:30左右，<font color="#FF0000">153由于磁盘被写满，挂掉，整个集群挂掉</font>
7. 9/18 15:00左右，重启152
8. 9/18 15:01左右，重启153， 集群恢复
9. 恢复后， 152/153 主题目录271.feeds_content_process_video.v1下数据全无
10. 通过工具查看，显示topic的offset 0 – 0 (oldest - latest)
11. 通过工具查看，显示consumer的消费信息，分别为：<font color="#0000FF">3,7</font>
12. 数据丢失!

#### 结论

通过查看kafka集群的配置，以及恢复前后的kafka server.log。结合前面所述的kafka broker备份及恢复机制，我们认为，导致本次数据丢失的根本原因在于：
1. 由于配置参数unclean.leader.election.enable导致不在ISR内的152在恢复后接管了topic，执行broker的启动流程(leader)
2. 153在启动后，成为follower，遵循follower的启动流程，删除了原本应该保留的

#### 分析

152由于首先恢复，进而成为目标主题的leader，作为leader，他会根据自身本地的HW信息，对本地log进行裁剪。同时，对于后续的follower的fetch message请求，进行响应的答复：
* 正常请求，返回目标offset的消息
* 异常请求，返回OffsetOutOfRangeException(请求的数据，超过leader本地数据的边界，153在恢复后，对两个分区分别请求offset3/7的数据, leader本地只有0/0)

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0014_kafka_data_loss_and_new_mechanism/img/07_leader_fetchResponse_OutOfRange.png)

```
leader 152上的日志：
[2017-09-18 15:00:24,878] INFO Recovering unflushed segment 0 in log 271.feeds_content_process_video.v1-1. (kafka.log.Log)
[2017-09-18 15:00:24,878] INFO Completed load of log 271.feeds_content_process_video.v1-1 with log end offset 0 (kafka.log.Log)
[2017-09-18 15:00:49,828] INFO Recovering unflushed segment 0 in log 271.feeds_content_process_video.v1-0. (kafka.log.Log)
[2017-09-18 15:00:49,828] INFO Completed load of log 271.feeds_content_process_video.v1-0 with log end offset 0 (kafka.log.Log)
[2017-09-15 14:25:05,894] INFO Truncating log 271.feeds_content_process_video.v1-1 to offset 0. (kafka.log.Log)

[2017-09-18 15:01:42,657] ERROR [Replica Manager on Broker 0]: Error when processing fetch request for partition [271.feeds_content_process_video.v1,1] offset 7 from follower with correlation id 0. Possible cause: Request for offset 7 but we only have log segments in the range 0 to 0. (kafka.server.ReplicaManager)
[2017-09-18 15:01:42,676] ERROR [Replica Manager on Broker 0]: Error when processing fetch request for partition [271.feeds_content_process_video.v1,0] offset 3 from follower with correlation id 0. Possible cause: Request for offset 3 but we only have log segments in the range 0 to 0. (kafka.server.ReplicaManager)
```
从上述152的回复后日志我们可以看出，152在成为leader后，首先基于本地HW，将数据裁剪到了0，之后收到了153的fetch request，并基于本地的offset信息，返回了`OffsetOutOfRangeException`。

153启动恢复后，成为follower，此时follower会基于自身HW，truncate本地log，在此之后，他会向leader(152)发起fetch message request：
1. 正常fetch response, 将消息append到本地log
2. 异常响应OffsetOutOfRange，将“dirty”数据，清理掉（将原本存在本地的3/7 offset数据当做脏数据清理掉）

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0014_kafka_data_loss_and_new_mechanism/img/08_follower_fetch_response_OutOfRange.png)

```
[2017-09-18 15:01:15,963] INFO Recovering unflushed segment 0 in log 271.feeds_content_process_video.v1-0. (kafka.log.Log)
[2017-09-18 15:01:15,964] INFO Completed load of log 271.feeds_content_process_video.v1-0 with log end offset 3 (kafka.log.Log)
[2017-09-18 15:01:31,533] INFO Recovering unflushed segment 0 in log 271.feeds_content_process_video.v1-1. (kafka.log.Log)
[2017-09-18 15:01:31,533] INFO Completed load of log 271.feeds_content_process_video.v1-1 with log end offset 7 (kafka.log.Log)
[2017-09-18 15:01:42,490] INFO Truncating log 271.feeds_content_process_video.v1-1 to offset 7. (kafka.log.Log)
[2017-09-18 15:01:42,496] INFO Truncating log 271.feeds_content_process_video.v1-0 to offset 3. (kafka.log.Log)

[2017-09-18 15:01:43,507] INFO Truncating log 271.feeds_content_process_video.v1-1 to offset 0. (kafka.log.Log)
[2017-09-18 15:01:43,514] ERROR [ReplicaFetcherThread-0-0], Current offset 7 for partition [271.feeds_content_process_video.v1,1] out of range; reset offset to 0 (kafka.server.ReplicaFetcherThread)

[2017-09-18 15:01:43,588] INFO Truncating log 271.feeds_content_process_video.v1-0 to offset 0. (kafka.log.Log)
[2017-09-18 15:01:43,596] ERROR [ReplicaFetcherThread-0-0], Current offset 3 for partition [271.feeds_content_process_video.v1,0] out of range; reset offset to 0 (kafka.server.ReplicaFetcherThread)

后续153成为partition1的leader：
[2017-09-18 15:06:08,795] INFO [ReplicaFetcherManager on broker 1] Removed fetcher for partitions [271.feeds_content_process_video.v1,1] (kafka.server.ReplicaFetcherManager)
```

####问题性质

kafka是不会保证在所有集群挂掉情况下的数据可用性，换言之，在这种场景下的数据丢失，<font color="#0000FF">是kafka已知的限制</font>

> For a topic with replication factor N, we will tolerate up to N-1 server failures without losing any records committed to the log.

来自：https://kafka.apache.org/documentation/#intro_guarantees

#### 解决方案

虽然我所遇到的问题，属于kafka的已知限制，但是，我们还是应当尝试尽量避免这类数据丢失的问题

* 确保集群broker的可用性
 * 不能出现集群N个broker挂掉的场景出现
* 调整unclean.leader.election.enable为false，不让152重启后接管
 * 可用性有一定损失
* 注意恢复后的主机启动顺序
 * 谁后挂，谁先起（让153先恢复）



### 补充说明

#### leader对于fetchRequest的处理

原本我是以为，对于最后一个follower (A)，在发起对m2的fetch request的时，由于此时leader(B)已经计算出HW应该更新到m2了，则此时返回给follower (A)的fetch response里面，写到的HW信息，应该已经是m2。但是经过参考文献[[1]](1)作者的提点，才明白，即使在这次fetch request过程中，leader已经更新了HW，但是返回的response里面，还是更新前的HW，也就是m1。具体可以参考kafka处理fetch response的代码：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0014_kafka_data_loss_and_new_mechanism/img/03_leaderHW_fetchResponse.png)


### 参考文献

[[1]:Kafka水位(high watermark)与leader epoch的讨论](http://www.cnblogs.com/huxi2b/p/7453543.html)