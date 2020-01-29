---
title: HDFS Lease(租约)逻辑
date: 2020-01-28 23:55:14
tags: 
	- Hadoop
	- HDFS
	- lease
---

HDFS对于同一个文件支持一写多读（write-once-read-many）。为了保持数据一致性，当一个客户端往HDFS某个文件写数据时， 其他客户端不允许同时写入。HDFS引入Lease(租约)机制来实现“独写”控制。 

本文基于hadoop-2.7.2版本对HDFS的租约机制进行整理分析

<!-- more -->

# Lease的组织形式

lease由NameNode(`LeaseManager`)负责维护。一个lease是一个client(由clientName决定，DFSClient_xxxx, DFSClient.java)可写的若干文件的凭证记录，他记录了该client(holder)对哪些文件具有写权限，同时，lease具有一定的独占期限（2s, `SoftLimit`）,如果该client未能在上一轮续约后2s内完成新的续约，该client拥有的可写权限将允许被其他用户抢占，同时，lease还具有一个失效期限，当client长时间未续约（60min，`HardLimit`），NameNode将负责从后台直接清理掉该lease，client将不再具有对之前任何文件的写权限。

简单来说，client要正常拥有对文件的写权限，就需要每隔2s，完成一次lease续约(renew lease)。如果在2s之后，还没有进行续约，则其他client可以进行抢占，但如果没有其他client抢占，该client还是拥有这份lease，同样还是对文件具有写权限。如果60min之后，client还没有对lease进行续约，那么，NameNode将视该lease过期，并对该lease进行清理，此时client不再具有对文件的写权限。

为判断一份lease是否有效，lease记录最近一次，该client申请续约的时间（`lastUpdate`)。一个Lease的结构组成如下：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0024_hdfs_lease_internal/img/01_lease_strucuture.png)

* holder: 记录了这是哪个client的租约信息
* lastUpdate: 记录上一次续租时间
* paths: lease包含的文件信息(这个client对哪些文件可写)

<font color=#0000FF>Lease是与client(holder)对应的，在后台NameNode上，每一个client有一份唯一的Lease与之对应。</font>在这份Lease中，记录了该client拥有写权限的所有文件信息(paths)。为了维护client <--> Lease <--> files之间的关系，方便分别从client的角度，从file的角度去访问、判断lease信息，NameNode(LeaseManager)使用了3个数据结构来完成相关管理工作：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0024_hdfs_lease_internal/img/02_lease_map_relation.png)

* leases: 用于管理client --> lease之的关系
* sortedLease: 按lastUpdate(最近一次续约时间），由早到晚排序的所有lease列表
* sortedLeasesByPath: 用于管理file(path) --> lease之间的关系信息

通过leases结构，可以找到某个client的lease信息，进而可以找到，他对哪些file有写权限；通过sortedLease, 从前往后遍历，可以判断哪些lease已经过期，需要进行超期清理；通过sortedLeaseByPath，可以判断当前某个file是归属于哪个lease，进而判断是哪个client对该file有写权限。

# Lease的创建(添加)

在发起对一个文件的持续写入之初，需要首先申请创建针对该文件的对应租约信息。如果该client在NameNode上已经有了对应的租约，则需要将当前的目标file添加到该租约中去，并更新最新续租时间。如果这是client第一次申请租约，则需要创建对应的租约，并将目标file添加到这个新创建的租约中。`LeaseManager`通过`addLease`接口完成上述逻辑，在`addLease`接口中，将针对性的调整`leases`, `sortedLease`, `sortedLeaseByPath`三个结构的内容。


client端，发起对文件的持续写入主要涉及与NameNode的3个RPC接口(service ClientNamenodeProtocol)：
* <font color=#0000FF>create</font>, 创建新文件，用于写入
* <font color=#0000FF>append</font>，已追加写的方式打开已有文件
* <font color=#0000FF>tuncate</font>，对当前文件执行阶段操作

## create

用户在创建新文件用于写入时，会通过`DistributedFileSystem::create(...)`调用，获取对应的输出流，进而执行后续的数据写入。在此过程中`DistributedFileSystem`会调用`DFSClient::create`，并最终在`DFSOutputStream`中调用`dfsClient.namenode.create(...)`，向NameNode发起create RPC调用。

NameNodeRpcServer执行create处理，在执行过程中，会首先通过`recoverLeaseInternal`来判断当前client是否可以获取到该file的lease。
1. 首先判断该file是否已有lease存在，
2. 如果没有，当然一切ok。
3. 如果有，是否过期。
4. 显然如果没过期，当前用户是无法抢占的。
5. 如果已过期，判断是否能够立马关闭当前文件，从而获取lease，
6. 如果不行，则发起lease recovery。发起lease recovery，意味着当前用户也无法立马获取到该文件的lease。（*<font color=#0000FF>关于`recoverLeaseInternal`的具体说明，参见后面2.4的章节</font>*）


当一切检查都通过， client满足获取lease的条件后，则创建client与file的lease，将其加入到`LeaseManager`的管理中

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0024_hdfs_lease_internal/img/03_create_function.png)

在调用`recoverLeaseInternal`时，如果未能立即获取到lease，将抛出异常。


## append

append用于追加写的方式打开一个已经存在的文件，对应逻辑与create基本类似。

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0024_hdfs_lease_internal/img/04_append_function.png)


## truncate

truncate与create/append不太一样，详细说明可参考：[HDFS truncate(HDFS-3107)](https://issues.apache.org/jira/browse/HDFS-3107)。NameNode在处理Truncate动作时，会根据实际truncate边界进行不同的处理，如果truncate的边界，正好在一个block的尾巴上(两个block的连接处)，不需要对某个block进行截断裁剪的话。则此时要做的事情，就是直接截断，更新block信息。但如果不是在边界上，需要对block中的数据进行截取，则会由NameNode发起一个BlockRecovery过程(`initializeBlockRecovery`)，根据truncate的设置，对block进行再定长，完成所有副本的truncate操作。

两种场景，对应了truncate的两个返回值:

1. return true。如果裁剪在边界上， truncate能够直接完成，并不会去获取lease信息，只确保能够拿到lease就ok，因为在truncate返回之后，truncate实际已经执行完成，无需持有lease。

2. return false。truncate没有直接完成，只是发起了block recovery动作，因此需要在block recovery的整个过程中持有lease，用户需等待truncate完成之后，重新append打开文件(再次获取lease)，进行数据再追加。

> When the truncate operation returns to the client the file <font color=#0000FF>may or may not be immediately available for write. **If the newLength happened to be on the block boundary, then the file can be opened for append immediately. Otherwise clients should wait for the truncate recovery to complete.**</font> During the recovery an attempt to append will result in <font color=#FF0000>RecoveryInProgressException</font>.

对于第1种情况：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0024_hdfs_lease_internal/img/05_truncate_function_01.png)

对于第2中场景：

由于需要在执行block recovery过程中，确保没有其他用户的持有lease，因此，需要添加lease操作:

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0024_hdfs_lease_internal/img/06_truncate_function_02.png)

如上图所示，如果truncate在block中间，则会由NameNode发起block recovery过程，在发起block recovery之前，会在`prepareFileForTruncate`中首先申请添加Lease。Block Recovery任务会随着DataNode的心跳响应异步的下发至DataNode，并执行。在DataNode完成Block Recovery之后，会向NameNode汇报结果(`commitBlockSynchronization`)，NameNode在此完成File的数据更新，并最终关闭文件，将Lease释放。(*<font color=#0000FF>具体的Block Recovery流程，可参见后续2.4.2章节描述。</font>*)


## recoverLeaseInternal

`recoverLeaseInternal`在整个Lease管理中的作用非常重要，他用于判断是否可以接管lease。
1. 当前用户clientA是否可以立马获取到对应文件的租约？
2. 如果当前file不在写入状态(`UnderConstruction`)，那显然，当前用户是可以拿到该文件的Lease的
3. 如果当前file处于UC状态，说明**<font color=#FF0000>可能</font>**正在有数据向该文件写入
 1. 如果是要强制抢占(**<font color=#0000FF>force==true</font>**)，此时不管太多，尝试直接接管(`internalReleaseLease`)
 2. 有数据写入(UC状态)，则判断该file的lease是否超期(`softLimit`)
   	1. 如果没有超期，显然clientA不能抢占lease
   	  * 要么此时文件已经处于recovery过程中，不能抢占(`RecoveryInProgressException`)
   	  * 要么是有前端clientB正在写文件，也不能抢占(`AlreadyBeingCreatedException`)
   	2. 如果clientB的lease已经超期了，那么就让clientA尝试直接接管(internalReleaseLease)

### internalReleaseLease

`internalReleaseLease`的任务是**<font color=#FF0000>尝试直接接管</font>**，如果能够直接接管，返回true。如果目前暂时无法直接接管，需要进行block recovery，则发起block recovery，并返回false。返回false，表示已经启动了lease recovery，没法直接接管，需要等待。其他情况，则抛出异常。

1. 直接接管(立刻拿到lease)的条件是，可以将当前file直接close掉。这里主要是看file中block的状态，
 1. 如果file的所有block都为completed，那显然是可以立马close的。
 2. 如果只有file的最后一个block是committed，其他都是completed状态，那进一步看看是否至少有一个datanode提交了最后一个block，保证满足了最小有效副本数，如果有了， 那也可以close了。否则就需要进行block recovery
 3. 另外还有一种情况，如果最后一个block不是commit，长度为0，并且没有DataNode汇报过该block的信息，这说明，client在写数据前，就故障了，此时可以将该block删除，并close掉，也可以直接接管
2. 如果最后一个Block处于**<font color=#0000FF>UNDER_CONSTRUCTION</font>**或者**<font color=#0000FF>UNDER_RECOVERY</font>**，已经有数据写入了，显然没法直接接管，需要通过block recovery，`uc.initializeBlockRecovery(...)` 对最后一个block的长度进行协商判定，确认之后，再将file close掉。这会是一个异步过程，无法直接接管。
3. 其他一些异常情况，则抛出异常


### Block Recovery过程

从上面`internalReleaseLease`的逻辑描述我们可以看出，在尝试直接接管的时候，存在当前文件的最后一个Block已经有数据写入的情况，3个Block副本的数据写入状态不一定一致，此时将由NameNode发起Block Recovery，Block Recovery流程大致如下：

1. NameNode在`internalReleaseLease`中启动BlockRecovery：
 1. 拿到一个新的`GnernationStamp`，作为recovery block使用的GS。`nextGenerationStamp(blockIdManager.isLegacyBlock(uc))`
 2. 将文件的所有权(lease)赋给recovery调用方：如果是用户抢占触发的，则所有权保持不变，还是之前的client，如果是强制抢占触发的，则所有权为本次强制抢占的调用方，如果是后台Lease超期清理触发的，则所有权为**<font color=#0000FF>NAMENODE_LEASE_HOLDER(HDFS_NameNode)</font>**
 3. 接着`initializeBlockRecovery`，
    * 设置该block进入Recovery状态，`setBlockUCState(BlockUCState.UNDER_RECOVERY)`
    * 从该block对应的3个副本所有datanode中选取其中之一，作为block recovery的发起方节点primary datanode
    * 将recovery指令加入目标primary DataNode的recovery指令队列:  `primary.getExpectedStorageLocation().getDatanodeDescriptor().addBlockToBeRecovered(this)`，待后续该DataNode的心跳过来时，NameNode将下发recovery指令

2. NameNode收到目标DataNode的心跳：`handleHeatbeat`
 1. 获取关联到该dn的recovery指令。`getLeaseRecoveryCommand(Integer.MAX_VALUE)`;
 2. 将recovery指导下发：`brCommand.add(new RecoveringBlock(...)`

3. DataNode接收NameNode的心跳响应，并处理相关指令：`processCommandFromActive`
 1. 执行recovery指令（case DatanodeProtocol.**<font color=#0000FF>DNA_RECOVERBLOCK</font>**）。`dn.recoverBlocks`
    * 首先通知该block的其他dn进入Replica Recovery，并获取各Replica的状态信息。`callInitReplicaRecovery` --> `datanode.initReplicaRecovery(rBlock)`
       1. 其他dn在收到`initReplicaRecovery`调用后，执行处理(`FsDatasetImpl::initReplicaRecovery`)。
       2. **<font color=#FF0000>首先会停止dn上当前的数据传输</font>**(`rip.stopWriter(xceiverStopTimeout)`)
       3. 之后汇总replica信息，并汇报给block recovery的发起方(primary datanode)
    * primary datanode汇总汇报的信息，并基于汇总信息对该Block定长，最终将定长结果同步更新到其他dn。`syncBlock(...)`
       1. 首先根据汇报的信息确定长度，
         * 如果有dn汇报该block已经finalized，那肯定已finalized为准。（`case FINALIZED:newBlock.setNumBytes(finalizedLength);`）
         * 如果没有finalized，则取小：`newBlock.setNumBytes(minLength)`;
         * truncate场景下，直接采用truncate的设置值：`newBlock.setNumBytes(rBlock.getNewBlock().getNumBytes())`;
       2. 之后将定长结果同步到所有dn。`r.updateReplicaUnderRecovery(...)`
       3. 同步好之后，将最终结果汇报给NameNode，告知NameNode根据汇报结果，close文件。`nn.commitBlockSynchronization(..., close = true, ...)`

4. NameNode收到DataNode的`commitBlockSynchronization`请求，执行最终的处理
 1. 根据汇报的信息，设置Block的GS，length等信息。`storedBlock.setGenerationStamp(newgenerationstamp);storedBlock.setNumBytes(newlength);`
 2. 执行文件的关闭动作。`closeFileCommitBlocks(...)`
    * 在执行关闭动作时，会释放该file的对应的lease。`finalizeINodeFileUnderConstruction` --> `leaseManager.removeLease`。这里的lease，可能是HDFS_NameNode(Lease清理触发)，也可能是原本持有该client的lease（clientB尝试抢占触发)，也可能是强行recoverLease的调用方（调用recoverLease触发)

**<font color=#FF0000>至此，整个Block Recovery结束，file上的Lease被清理，且文件关闭，长度被确定。</font>**

***Q1. 如果Block Recovery过程中，原来的clientA恢复过来了，会出现什么情况？***

恢复过来也没有用，进入Block Recovery状态后，DataNode就会将之前的写入给断掉，并汇报最终状态到NameNode，NameNode会最终将文件close掉，释放文件上的Lease。因此即使ClientA恢复过来，也无法继续向文件写入数据，文件最终一定会到close状态。


***Q2. Block Recovery过程如果时间过长，在此过程中，是否还能被其他人抢占？***

HDFS_NameNode作为Lease持有者，执行恢复也好， 强制执行`recoverLease`的调用方ClientB作为持有者也好，甚至是主动抢占时触发Block recovery，保持原持有者ClientA作为持有者也好。如果Block Recovery没有在2s内完成，则也是可以被抢占的。只是这时候的抢占没有意义。原因是，再度进入internalReleaseLease逻辑，又会认为还需进行block recovery，进而又会发起新的一轮Block Recovery。只要上一轮的Block Recovery结束，NameNode处理`commitBlockSynchronization`，完成文件的close（`finalizeINodeFileUnderConstruction`），后续的抢占，就不再属于Block Recovery过程的抢占，而是正常抢占。


# Lease的续签

client在调用上述接口完成Lease申请之后，需要每隔2s，执行一次续签动作，否则对应的Lease将会失效。为此，在client端调用append/create之后，都将触发client端的LeaseRenewer线程，由`LeaseRenewer`线程不停的定时执行续签动作(`namenode.renewLease`)。

在DFSClient执行create/append，完成对应的create/append RPC调用之后，都将触发启动LeaseRenewer线程(`beginFileLease`)。LeaseRenewer是一个单例线程，会在第1次被调用时启动，之后不断的执行renew，他会判断对应的DFSClient的Lease当前时间距离上一轮续租是否超过1/2 SoftLimit(1s)，如果超过，则发起下一轮续租请求。


***<font color=#FF0000>从Lease续签的动作我们可以看出，续签是以Lease，也就是以client为单位的，不涉及对其中某个文件的续租，而是整体的续租。具体续租了哪些文件，这完全由NameNode(LeaseManager)中，该Lease对应的文件信息来决定。如果，通过其他调用，修改了NameNode上，该Lease对应的文件信息，例如移除了其中的某个文件file1。那么，即使该Client端还在不断的续租，而实际上，Client对file1的写权限，已经没有了。</font>***


# Lease的主动释放

对于申请了lease的三个接口：create/append/truncate，都需要有与之对应的Lease主动释放。其中，从上面truncate的逻辑描述我们可以看出，如果truncate在block边界。实际并未申请Lease，因此也不需要释放Lease。而如果truncate在block中间，从上面的流程我们也可以看到，当DataNode完成Block Recovery之后，会有NameNode完成Lease的释放(`finalizeINodeFileUnderConstruction`中)。

而对于调用了create/append的用户，则需要主动的调用DFSOutputStream::close/abort，在close/abort时，会调用**<font color=#0000FF>complete</font>** RPC接口，完成Lease的释放

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0024_hdfs_lease_internal/img/07_complete_rpc.png)


# 结束Lease续签

`LeaseRenewer`会记录每个DFSClient究竟打开了哪些文件，当`DFSOutputStream::close`时，会将打开的文件一并从`LeaseRenewer`中移除(`endFileLease`)，因此。当`LeaseRenewer`发现DFSClient上没有打开的文件时(`dfsc.isFilesBeingWrittenEmpty()`)，将不再对DFSClient发起Lease续签(`dfsclients.remove(dfsc)`)

# Lease的校验

当用户通过create/append发起对文件的写入后，在持续不断的写入block过程中，涉及到若干RPC调用，其中与NameNode交互，涉及到元数据变更的调用都将执行Lease的校验，其包括：

* `addBlock`：获取一个新的Block，用于写入(create后第一个Block的获取也使用该接口）
* `getAdditionalDatanode`: 建立pipeline过程中出现异常，申请额外的DataNode用于构建数据写入的pipeline
* `abandonBlock`：由于某些原因(datanode故障）导致对指定block的写入失败，client丢弃该block，之后尝试获取新的Block
* `complete`：完成对指定文件的写操作，关闭文件
* `fsync`: 在NameNode上持久化文件的元数据（block list）信息（在DFSClient close)需要使用

NameNode在处理上述RPC调用时，都将执行Lease的校验(`LeaseManager::checkLease`)，校验过程：

1. 校验的文件是否存在(inode == null). 
2. 校验目标是否为文件(非目录)(!inode.isFile())
3. 校验的文件是否处于可写状态(!file.isUnderConstruction())
4. 校验的文件是否处于待删除状态(isFileDeleted(file))
5. **<font color=#0000FF>校验文件Lease的实际持有者，与当前目标是否一致</font>**`(holder != null && !clientName.equals(holder))`
 * 在这一步校验中，并没有校验File当前的Lease是否已经超过`SoftLimit`。因此对于Lease是否有效，我们可以认为，只要Lease存在，Lease就是有效的，只是超过SoftLimit，是允许其他client抢占。

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0024_hdfs_lease_internal/img/08_addBlock_rpc.png)

在上述校验过程中出现失败，意味着对应的File已经进入了Block Recovery状态，由其他人接管了该文件。而在Block Recovery状态时，DN已经停止了之前的数据写入，并在停止写入之后，对Block进行了定长判断。定长被认的数据，就是"真实"写入的数据，定长被丢掉的数据，就是未写入的数据。数据长度的判定，并不发生在原client `checkLease`失败的地方，而是发生在接管，BlockRecovery执行的过程中。`checkLease`的失败，只是最终结果，实际上，数据传输已在Block Recovery的时候被停止掉。


# Lease的抢占(SoftLimit超时)

当clientA对他注册在NameNode上的Lease超过2s没有续租时，其他用户(ClientB)就可以对其中的file lease进行抢占。抢占的过程，就发生在create/append/truncate调用时，由`recoverLeaseInternal`负责执行，其具体过程参考***<font color=#0000FF>2.4.1 recoverLeaseInternal章节的说明</font>***。抢占可以直接完成，此时clientB直接拥有了对file的写权限，继续自己的后续写入。抢占也可能不直接完成，此时`recoverLeaseInternal`会触发block recovery。而block recovery完成之后，文件处于close状态， clientB可再次尝试申请Lease。



# Lease的超期清理(HardLimit超时)

`LeaseManager`还有一个`HardLimit(1h)`，`LeaseManager`中的monitor线程，会每隔2s启动一次，判断NameNode上的Lease是否有超过`HardLimit(checkLeases())`，如果有，则执行超期清理工作。
1. 执行清理工作主要基于sortedLeases来完成，由于sortedLeases已经是根据续约时间排好序的，因此只需要从sortedLeases头部取出Lease判断是否超期，如果头部的Lease都没有超期`(!leaseToCheck.expiredHardLimit())`，则后续的Lease都不会超期。
2. 对于超期Lease的处理，`LeaseManager$Monitor`将拿出Lease中所有的file，因为Lease超期，意味着Lease中的所有文件都要释放掉。
3. 针对每个File，`LeaseManager$Monitor`同样是依赖于`internalReleaseLease`，完成超期释放动作：
a. 如果可以直接释放，则直接释放。
b. 如果不可以直接释放，则通过Block Recovery完成释放，此时Block Recovery释放的Lease持有者为(HDFS_NameNode)。
4. 如果`internalReleaseLease`的处理遇到异常，则`LeaseManager$Monitor`会强行将对应的Lease删除`(removeLease(leaseToCheck, p);)`


# Lease的强制清理

有2种情况会执行Lease的强制清理：
1. 删除该文件(包括删除对应的目录)
2. 执行recoverLease调用


## 删除文件(或者目录)

当文件被删除时，该文件对应的Lease将强制被删除掉。当有目录被删除时，该目录上所有文件的Lease都将一并删除掉(`removeLeaseWithPrefixPath`)

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0024_hdfs_lease_internal/img/09_delete_function.png)

## 强制释放Lease

**<font color=#FF0000>NameNode提供了recoverLease RPC调用来强制释放一个client持有某个file的Lease，无论这个Lease是否在效期内(SoftLimit, 2s)。</font>**recoverLease达到的效果：
1. return true，直接完成lease释放，文件已进入close状态
2. return false，启动Block Recovery，进行异步的lease释放，待Block Recovery完成，文件将进入close状态，lease释放。
3. 抛出异常，强制释放都出现问题@.@...

***<font color=#0000FF>调用recoverLease并不意味着立马完成close，如果返回true是立马完成， 如果返回false，还得等待，知道block recovery异步完成，文件close。</font>***

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0024_hdfs_lease_internal/img/10_recoverLease.png)



# Lease的变更

rename不会丢失Lease，他会一并对rename涉及的所有文件的Lease做统一调整(`changeLease`)，`changeLease`会将原始前缀为src的所有file找出来(`findLeaseWithPrefixPath`)，并将这些目标file的Lease调整变更为指定dst前缀。

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0024_hdfs_lease_internal/img/11_rename_function.png)


# 总结

1. Lease是client对某些文件是否具有可写权限的凭着，Lease是针对某个client的，一个Lease包含该client所有可写文件的信息。
2. Lease的管理及校验(文件可写权限校验）只发生在client与NameNode之间，与DataNode无关联
3. client续租Lease是对整个Lease进行续租，**<font color=#0000FF>不是针对某个文件的可写权限进行续租。</font>**一次续租Lease，究竟续租了client对哪些文件的可写权限，完全是依赖于NameNode记录的这份Lease中包括了哪些文件。**<font color=#0000FF>如果通过一定手段(`recoverLease`)强行剥夺了clientA对file1的lease权限，即使clientA在不断的续租lease，由于NameNode记录的clientA的lease中，已不包含file1信息，因此clientA也不再拥有对file1的写权限(租约）</font>**
4. `recoverLease`只保证能立即剥夺原持有者clientA对file1的写权限，但不能保证file1立马进入最终状态(closed)，在某些情况下(最后一个Block有数据写入时），需要执行最后一个Block的恢复(Block Recovery)，这个过程主要是完成最后一个Block的长度判定，并最终在NameNode上完成file的关闭。
5. 一个文件可写，就处于UnderConstruction，对应的，这个文件是未关闭的`(isClosed == false)`，对应的一定具有某个Lease包含这个文件。换句话说不存在某个文件可写，但是却没有任何一个Lease包括这个文件。
