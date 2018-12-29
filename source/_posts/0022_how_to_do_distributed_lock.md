---
title: 如何实现分布式锁
date: 2018-11-18 20:58:14
tags: 
	- 分布式
	- redis
---

分布式锁服务在分布式系统中是一个非常通用的需求。互联网行业有基于Zookeeper实现分布式锁服务的方案，也有提出基于Redis实现分布式锁服务的方案。企业级应用方面，开源Linux上，Redhat Linux HA套件中提供了[DLM(Distributed Lock Manager)](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/high_availability_add-on_overview/ch-dlm)，商用操作系统OpenVMS也提供了[DLM](http://neilrieck.net/docs/openvms_notes_DLM.html)。下面的这篇文章，是我于18年初的时候读到的，最近又重读一遍，作为回顾总结。翻译自Martin Kleppmann的博文：[How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)

<!-- more -->

作为[本书](http://dataintensive.net/)研究的一部分，我在[Redis](http://redis.io/)网站上遇到了一种名为[Redlock](http://redis.io/topics/distlock)的算法。该算法声称基于Redis实现了容错的分布式锁（或者更确切的说，是[租赁](https://pdfs.semanticscholar.org/a25e/ee836dbd2a5ae680f835309a484c9f39ae4e.pdf)），并且在网页上要求来自分布式系统人员的反馈。本能性的在我脑海中引发了对该算法一些警惕，所以我花了一些时间思考他，并写下了这些笔记。

由于已经有超过[10个RedLock的独立实现](http://redis.io/topics/distlock)，我们不知道谁已经依赖这个算法，我认为将我的笔记公开出来是有价值的。我不会涉及Redis的其他方面，其中一些方面已经在[其他地方](https://aphyr.com/tags/Redis)被批评过了。

在我深入Redlock的细节之前，我首先必须声明，我非常喜欢Redis，并且过去我也非常成功的将其应用到生产环境中。我认为他非常适合运用到你想要在服务器之间共享一些瞬态、近似、快速变化数据的场景，以及如果你因为某种原因丢失数据，但这种数据丢失影响不大的场景。例如，一个很好的例子就是基于IP地址来记录请求数量(用于流控目的)以及基于用户ID汇总IP地址集合。

然而，Redis已经逐渐进入数据管理领域，这些领域具有更强的一致性和持久性预期 -- 这让我很担心，因为这并不是Redis的设计目标。

可以说，分布式锁就是其中一个领域。让我们更详细的检视他。

# 你用分布式锁来做什么

分布式锁的目的是确保在可能尝试执行相同工作的多个节点中，只有一个节点会执行该工作（至少每次工作仅有一个节点执行）。这个工作可能是向共享存储系统写入一些数据，或者是执行一些计算，或者是调用一些外部API等诸如此类的。站在更高的层面，为什么在一个分布式系统内你需要分布式锁，可能有两个原因：[或为了更高效，或为了正确性](http://research.google.com/archive/chubby.html)。通过质问在分布式锁失败的场景下，到底会引起什么问题，你可以区分这两类场景：

* **为了更高效**：通过使用锁，使得你不必要对相同的事情做两遍（例如：一些代价很大的计算工作）。如果分布式锁失败，而两个节点最终完成相同的工作，结果是成本略有增加（最终为AWS支付的费用比你原本预期的要多5美分）或稍有不便（例如：用户最终两次收到相同的电子邮件通知）。

* **为了正确性**：通过分布式锁可以防止并发进程踩到彼此的脚趾，进而扰乱整个系统状态。如果分布式锁失败，而两个节点同时处理了同一条数据，则结果是文件损坏，数据丢失，永久性系统不一致，给予患者的药物计量错误以及一些其他严重的问题。

这两类都是需要分布式锁的合适案例，但你需要非常清楚，你正在处理的是上述两类中的哪一类。

我会争辩，如果你仅仅是为了提高效率而使用分布式锁，则没有必要承担Redlock的成本和复杂性，他要运行5个Redis服务器，并通过检查多数来获取你的分布式锁。你最好只使用一个Redis实例，也许在配一个异步复制到备份实例，用于应对主实例崩溃的场景。

如果你是使用Redis单实例，当然，在你Redis节点突然掉电的情况下， 你会丢失一些分布式锁。但如果你使用分布式锁仅仅是为了更高效，而且节点崩溃事件并非经常发生，这就不是什么大问题。这类“不是什么大问题”的场景正式Redis合适的地方。至少如果你依赖于单个Redis实例，那么每个分析系统的人都知道，那个分布式锁是起到近似的作用，并且仅用于非关键目的。

而另一方面，具有5个副本和多数表决权的Redlock算法，乍一看，好像它适用于分布式锁应对*正确性*很重要的场景。我将在以下各章节中论证他不适合这个目的。对于本文的其余部分，我们将假设，你使用分布式锁是为了保障正确性，并且两个不同的节点同时持有相同的锁，这种情况是一个很严重的错误。


# 通过锁来保护资源

让我们暂时搁置Redlock的实现细节，先来讨论一下如何使用分布式锁（与所使用锁的具体实现算法无关）。需要记住的很重要的一点是，分布式系统中的锁与多线程应用程序中的互斥锁是有区别的。因为系统中不同节点和网络都可以以各种方式失败，因此相较而言，分布式系统中的锁是一个更复杂的野兽。

举例来说，假设你有一个应用，他其中有一个客户端需要向共享存储(例如HDFS或者S3)更新一个文件。客户端先是拿到锁，然后读取文件，接着做些修改，然后将修改后的文件写回共享存储，并最终释放锁。这个锁能够防止两个client同时执行读出-修改-写回这个周期，两个client同时执行这个周期将会导致更新的丢失。代码可能如下所示：

```java
// THIS CODE IS BROKEN
function writeData(filename, data) {
    var lock = lockService.acquireLock(filename);
    if (!lock) {
        throw 'Failed to acquire lock';
    }

    try {
        var file = storage.readFile(filename);
        var updated = updateContents(file, data);
        storage.writeFile(filename, updated);
    } finally {
        lock.release();
    }
}
```
不幸的是，即使您拥有完美的分布式服务，上面的代码也会被破坏。 下图演示了如何导致最终结果是数据被破坏：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0022_how_to_do_distributed_lock/img/01_unsafe-lock.png)

在上面的例子中，拿到锁的客户端，在拿着锁的同时被暂停了一段时间 -- 例如，因为垃圾收集器(GC)启动了。分布式锁有一个超时（即，他是一个租约），这总是一个好主意（否则，一个崩溃的客户端可能永远持有一把锁且永远不会释放他）。然而，如果GC暂停的足够长，长到超过租约过期时间，而客户端却没有意识到他已经过期了，他会继续执行，并作出一些不安全的更新。

这个bug并非仅仅是理论上的，HBase曾经有过[这个问题](http://www.slideshare.net/enissoz/hbase-and-hdfs-understanding-filesystem-usage)。一般情况下GC暂停都非常短，但是，“停止世界”的GC暂停有时候会持续[几分钟](http://blog.cloudera.com/blog/2011/02/avoiding-full-gcs-in-hbase-with-memstore-local-allocation-buffers-part-1/) -- 而这对于租约过期而言已经足够长了。甚至是所谓的，像HotSpot JVM的CMS这样的“并发”GC也不能与应用程序代码完全保持并行执行 -- 甚至他们也[需要不时的停止世界](http://mechanical-sympathy.blogspot.co.uk/2013/07/java-garbage-collection-distilled.html)。

你不能通过在写回存储之前插入一个分布式锁是否过期的检测来修复这个问题。请记住，GC可以在任何时间点暂停正在运行的线程，包括对你来说最不舒服的时间点(在最后检查完成，准备执行写入操作之间)。

如果你因为你使用的编程语言运行时没有长时间的GC暂停而得意，那么我只能告诉你，你的进程可能被暂停的原因还有很多。也许你的进程尝试读取尚未加载到内存中的地址，因此它会出现页面错误并暂停，直到从磁盘加载页面为止。也许您的磁盘实际上是EBS，因此读取变量无意中变成了亚马逊拥塞网络上的同步网络请求。也许还有许多其他进程争用CPU，并且您在调度程序树中遇到了一个[黑色节点](https://twitter.com/aphyr/status/682077908953792512)。 也许有人不小心将SIGSTOP发送到了这个进程。等等诸如此类的，你的进程将被暂停。

如果你仍然不相信我的进程会被暂停的说法，那么考虑换一个角度，文件数据写入的请求，在到达存储服务之前可能会在网络中被延迟。诸如以太网和IP之类的分组网络可能会*任意*延迟数据包，并且[他们会](https://queue.acm.org/detail.cfm?id=2655736)：在[GitHub的一个著名事件](https://github.com/blog/1364-downtime-last-saturday)中，数据包在网络中延迟了大约90秒。这意味着应用程序进程可以发送写入请求，而这个请求可能在租约已经过期的1分钟之后才到达存储服务器。

即使在管理良好的网络中，也会发生这种情况。你根本无法对时序做出任何假设，这就是为什么上面的代码根本不安全的原因，无论你使用什么分布式锁服务。


# 使用防护栏使锁安全

解决此问题的方法实际上非常简单：你需要在向存储服务发起的每个写入请求中包含一个防护令牌。在此场景下，防护令牌只是每次客户端拿锁的时候递增(例如，由锁服务递增)的一个数字。如下图所示：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0022_how_to_do_distributed_lock/img/02_fencing-tokens.png)

客户端1拿到租约，并获得33号令牌，但是之后他进入一个长的暂停，并且租约过期。客户端2拿到租约，获得令牌34（这个数字总是递增），接着向存储服务发送他的数据写入请求，在请求中包含令牌34。之后，客户端1恢复过来，并向存储服务发送他的包含令牌33的写入请求。然而，由于存储服务记录了他已经处理了一个带有更大令牌号(34)的数据写入请求，因此，他会拒绝包含令牌33的请求。

注意，这里要求存储服务主动来校验令牌，并且将那些带有小编号令牌的数据写入请求给拒绝掉。一旦你掌握了诀窍，这其实并不难。假设锁服务能够生成严格单调递增的令牌，这将使得锁更安全。举例来说，如果你使用Zookeeper作为锁服务，你可以使用`zxid`或者znode的版本号来作为防护令牌，此时，你的锁使用方式是安全的。

然而，Redlock没有任何生成防护令牌的工具，这是我们检视Redlock发现的第一大问题。算法并没有生成任何在每次客户端拿锁时都保证会递增的数字。这意味着即使算法完美无缺，使用也不安全，因为在客户端暂停或其数据包被延迟的情况下，你无法阻止客户端之间的竞争条件。

同时，对我来说，如何更改Redlock算法进而可以开始生成防护令牌也不是那么显而易见的事情。算法使用的唯一随机值，并不能提供所需的单调性。仅仅在一个Redis节点上保留计数器是不够的，因为该节点可能会挂掉。将计数器保留在几个节点上意味着他们会不同步。你可能需要一个共识算法才能生成防护令牌（如果仅仅是增加一个[递增计数器](https://twitter.com/lindsey/status/575006945213485056)是很简单的事情）

# 用时间来解决共识

Redlock无法生成防护令牌的事实，已经足以成为在为正确性而使用分布式锁的场景下不使用它的理由。但他还有一些值得我们讨论的更进一步的问题。

在学术界，这类算法最实际的系统模型是[具有不可靠检测器的异步模型](http://courses.csail.mit.edu/6.852/08/papers/CT96-JACM.pdf)。简单来说，这意味着算法不会对时序做出任何假设：进程可能暂停任意长的时间，数据包可能在网络中被任意延迟，时钟可能有任意错误的 -- 然而算法仍然可以做到正确工作。考虑我们上面讨论到的，这些都是非常合理的假设。

算法可以使用时钟的唯一目的是生成超时，以避免在节点挂掉时会永远等待。但是，超时不需要是精准的：仅仅因为请求超时，并不意味着另一个节点肯定挂掉了 -- 它也可能是因为网络中存在大的延迟，或者你的本地时钟是错的。当被用作故障检测器时，超时只是猜测出现了问题。 （如果可以的话，分布式算法将完全没有时钟，但随后的[协商将变得不可能](http://www.cs.princeton.edu/courses/archive/fall07/cos518/papers/flp.pdf)。拿到锁就像是CAS操作，这需要[达成共识](https://cs.brown.edu/~mph/Herlihy91/p124-herlihy.pdf)。）

请注意，Redis[使用gettimeofday](https://github.com/antirez/redis/blob/edd4d555df57dc84265fdfb4ef59a4678832f6da/src/server.c#L390-L404)，而不是[单调时钟](http://linux.die.net/man/2/clock_gettime)来确定key的[到期时间](https://github.com/antirez/redis/blob/f0b168e8944af41c4161249040f01ece227cfc0c/src/db.c#L933-L959)。gettimeofday的手册页明确表示它返回的时间会受到系统时间不连续跳跃的影响 - 也就是说，它可能会突然向前跳几分钟，甚至会从当前时间往回跳（例如，如果时钟由NTP同步，由于它与NTP服务器时间差异太大，或者如果时钟由管理员手动调整）。因此，如果系统时钟出现奇怪的现象，很容易发生Redis的key的到期时间比预期的时间快的多或者慢的多。

对于异步模型中的算法，这不是一个大问题：这些算法通常是在[不对时序做任何假设的情况下](http://www.net.t-labs.tu-berlin.de/~petr/ADC-07/papers/DLS88.pdf)，来确保算法的安全属性始终能得到保证。仅仅只有节点存活这个属性依赖于超时或者其他的故障检测手段。简单来说，这意味着即使系统中的时间发生各类异常（进程暂停，网络延迟，时钟向前和向后跳跃），算法的性能可能下降，但算法永远不会做出错误的决定。

然而，Redlock却不是这样。他的可靠性依赖很多时序假设：他假设所有Redis节点在key到期之前都持有key维持一个合适的时间长度；与到期时间间隔相比，网络延迟都比较小；并且进程的暂停时间比到期时间间隔要小得多。

# 用恶意的时序来推翻Redlock算法

让我们来看一些示例，证明Redlock的可靠性依赖于时序的假设。例如，系统有5个Redis节点(A,B,C,D和E)，并且有2个客户端(1和2)。如果一个Redis节点的时钟向前跳跃了，会发生什么事情呢？

1. 客户端1在A,B,C节点上获取到锁。由于网络原因，D和E没有被访问到。
2. C节点的时钟向前跳跃，导致他上面的锁过期了。
3. 客户端2在C,D,E节点上拿到锁，由于网络原因，A和B没有被访问到。
4. 客户端1和2都认为自己拿到了锁。

如果C节点在将锁的信息持久化到磁盘之前挂掉了，并立马又重启，也会出现类似的问题。正是出于这个原因，Redlock的文档建议挂掉的节点延迟重启，至少要等集群中最长的锁到期期限过了之后再重启。然而，这个延迟重启又依赖于严格的时间度量，如果又有时钟向前跳跃，延迟重启之后可能还是有问题。

好吧，也许你会认为时钟跳跃不太可能，因为你非常有信心能够正确配置NTP，让他仅仅能使时钟做一些滑动（而不是跳跃）。如果是这样的话，那让我们看一个示例，看进程暂停如何能导致算法失败：

1. 客户端1向A,B,C,D,E发起拿锁请求。
2. 当返回客户端1的响应还在途时，客户端1进入停止世界GC
3. 所有Redis节点上的锁都过期失效
4. 客户端2在节点A,B,C,D,E上拿到锁
5. 客户端1完成GC，并收到所有Redis节点的，表明他已成功拿到锁的响应(当进程被暂停时，这些响应已经到达客户端1的内核网络缓存中)。
6. 客户端1和客户端2均认为自己持有锁。

这里请注意，即使Redis是用C实现的，并且没有GC，但是此时也没法帮到我们：任何客户端系统，如果他可能会经历GC暂停的话，都可能碰到这个问题。在客户端2拿到锁之后，你只能通过阻止客户端1执行任何它在拿到锁之后的操作，才能保证算法的可靠，例如使用防护栏措施。

一个长时间的网络延迟，也可以达到进程暂停的效果。他或许依赖于你的用户TCP超时 -- 如果你让这个超时比Redis的TTL短很多，也许网络延迟包会被忽略，但我们必须深入了解TCP的实现，才能确保这一点。同样，因为涉及到超时，我们又回到精确度量超时这个问题上来了。

# Redlock的同步假设

上面的这些事例都表明Redlock仅能在你假设系统为同步系统模型时才能正确工作 -- 这个模型有如下属性：

1. 有限的网络延迟（你能保证网络包总是在某个最大延迟之内到达）
2. 有限的进程暂停（换句话说，强实时性要求，你或许仅会在汽车安全气囊系统中才会遇见这类要求）
3. 有效的时钟错误（祈祷你不会从一个[有问题的NTP服务器](http://xenia.media.mit.edu/~nelson/research/ntp-survey99/)上拿到时间）

需要注意的是，同步模型并不意味着精确的同步时钟：他的意思是你假设了在网络延迟、进程暂停和时钟漂移上的[已知的固定上限](http://www.net.t-labs.tu-berlin.de/~petr/ADC-07/papers/DLS88.pdf)。

在表现正常的数据中心环境下，大多数时候时序假设都是成立的 -- 这被认为是[部分同步系统](http://www.net.t-labs.tu-berlin.de/~petr/ADC-07/papers/DLS88.pdf)。但是，这样就足够好了吗？一旦时序假设被破坏，Redlock就可能变的不可靠，例如，一个客户端在另一个客户端超时之前，就获得了租约。如果你依赖于你的锁服务来达到正确性，“大多数时候”是不够的，你得*始终*正确。

这里有很多证据表明，对于大多数实际系统环境，假设其为同步系统模型是不安全的。提醒自己关于GitHub的[90秒数据包延迟](https://github.com/blog/1364-downtime-last-saturday)事故。这也难怪为什么Redlock无法通过[Jepsen](https://aphyr.com/tags/jepsen)测试。

另一方面，针对部分同步系统模型（或者是带有故障检测的异步系统模型）而设计的一致性共识算法有机会派上用场了。Raft, Viewstamped Replication, Zab以及Paxos都是这类算法。这类算法都必须放弃所有时序假设。这很难：假设网络、进程、时钟比他们实际上更可靠是如此诱人。但是，在分布式系统的混乱现实中，你必须非常小心你的假设。

# 结论

我认为Redlock算法是一个糟糕的选择，因为他“不伦不类”：如果是出于效率的考虑，他显的过于笨重且代价太大，而对于依赖于锁服务追求正确的场景而言，他又不足够安全。

实际上，这个算法对于时序和系统时钟做了危险的假设(基本上假设了一个带有有限网络延迟和有限操作执行时间的同步系统)，如果不满足这些假设，他就变的不可靠。此外，他还缺乏用于生成防护令牌的手段（防护令牌可以保护系统免受网络长时间延迟或进程被暂停带来的影响）。

如果你需要锁仅仅是为了尽力而为（仅作为效率优化，而不是为了确保正确性），我建议坚持使用Redis的简单单节点锁算法（通过如果不存在就设置的方式来拿到锁，原子性的如果匹配就删除来释放锁），并在你的代码中清楚的注释这里的锁只是近似的作用，有时可能会失败。不要费心去设置一个5个Redis节点的集群。

另一方面，如果你真的需要通过锁服务来保证正确性，请不要使用Redlock。取而代之，请使用类似于[Zookeeper](https://zookeeper.apache.org/)的合适的一致性共识系统，也许可以通过使用[Curator recipes](http://curator.apache.org/curator-recipes/index.html)之一来实现一个锁服务。(至少，也可以使用具有合理事务保证的数据库。)并且请在所有受锁保护的资源访问时，强制使用防护令牌。

正如我在开篇时说的，如果你能正确使用Redis，Redis是一个非常优秀的工具。以上的描述都不会减少Redis在他预期目标领域的有用性。[Salvatore](http://antirez.com/)多年来一直致力于该项目，Redis的成功当之无愧。但是，每一种工具都有自身局限性，了解他们并进行相应的规划非常重要。

如果你想了解更多信息，我在[我书的第8章和第9章](http://dataintensive.net/)中更详细地阐述了这个主题，我书的早期发布现在可以从O'Reilly找到。（上面的图都取自我的书。）如果想学习如何使用Zookeeper，我推荐[Junqueira和Reed的书](http://shop.oreilly.com/product/0636920028901.do)。如果想更好的了解分布式系统理论，我推荐[Cachin，Guerraoui和Rodrigues的教科书](http://www.distributedprogramming.net/)。

感谢[Kyle Kingsbury](https://aphyr.com/)，[Camille Fournier](https://twitter.com/skamille)，[Flavio Junqueira](https://twitter.com/fpjunqueira)和[Salvatore Sanfilippo](http://antirez.com/)审阅本文的草稿。 当然，任何错误都在我。

**2016年2月9日更新**：Redlock的原作者Salvatore对本文发表了[反驳](http://antirez.com/news/101)（另见[HN讨论](https://news.ycombinator.com/item?id=11065933)）。 他提出了一些好的观点，但我坚持我的结论。 如果我有时间，我可以在后续帖子中详细说明，但请形成您自己的意见 - 并且请参考下面的参考资料，其中许多已获得严格的学术同行评审（这与我们的任何博客文章不同）。

# 参考资料

[1] Cary G Gray and David R Cheriton: “Leases: An Efficient Fault-Tolerant Mechanism for Distributed File Cache Consistency,” at 12th ACM Symposium on Operating Systems Principles (SOSP), December 1989. doi:10.1145/74850.74870

[2] Mike Burrows: “The Chubby lock service for loosely-coupled distributed systems,” at 7th USENIX Symposium on Operating System Design and Implementation (OSDI), November 2006.

[3] Flavio P Junqueira and Benjamin Reed: ZooKeeper: Distributed Process Coordination. O’Reilly Media, November 2013. ISBN: 978-1-4493-6130-3

[4] Enis Söztutar: “HBase and HDFS: Understanding filesystem usage in HBase,” at HBaseCon, June 2013.

[5] Todd Lipcon: “Avoiding Full GCs in Apache HBase with MemStore-Local Allocation Buffers: Part 1,” blog.cloudera.com, 24 February 2011.

[6] Martin Thompson: “Java Garbage Collection Distilled,” mechanical-sympathy.blogspot.co.uk, 16 July 2013.

[7] Peter Bailis and Kyle Kingsbury: “The Network is Reliable,” ACM Queue, volume 12, number 7, July 2014. doi:10.1145/2639988.2639988

[8] Mark Imbriaco: “Downtime last Saturday,” github.com, 26 December 2012.

[9] Tushar Deepak Chandra and Sam Toueg: “Unreliable Failure Detectors for Reliable Distributed Systems,” Journal of the ACM, volume 43, number 2, pages 225–267, March 1996. doi:10.1145/226643.226647

[10] Michael J Fischer, Nancy Lynch, and Michael S Paterson: “Impossibility of Distributed Consensus with One Faulty Process,” Journal of the ACM, volume 32, number 2, pages 374–382, April 1985. doi:10.1145/3149.214121

[11] Maurice P Herlihy: “Wait-Free Synchronization,” ACM Transactions on Programming Languages and Systems, volume 13, number 1, pages 124–149, January 1991. doi:10.1145/114005.102808

[12] Cynthia Dwork, Nancy Lynch, and Larry Stockmeyer: “Consensus in the Presence of Partial Synchrony,” Journal of the ACM, volume 35, number 2, pages 288–323, April 1988. doi:10.1145/42282.42283

[13] Christian Cachin, Rachid Guerraoui, and Luís Rodrigues: Introduction to Reliable and Secure Distributed Programming, Second Edition. Springer, February 2011. ISBN: 978-3-642-15259-7, doi:10.1007/978-3-642-15260-3

