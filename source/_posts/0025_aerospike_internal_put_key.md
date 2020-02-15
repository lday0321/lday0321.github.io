---
title: Aerospike内部逻辑：从Record Put说起
date: 2020-02-15 21:56:14
tags: 
	- Aerospike
	- NoSQL
---

[Aerospike](https://github.com/aerospike)是一个分布式的Key-Value存储系统，广泛应用于广告行业的个性化推荐及实时竞价系统中。其主要特点是采用混合存储架构，数据索引信息存储在内存中，而数据本身则存储在硬盘上。Aerospike可绕过文件系统，独立管理写入磁盘的数据，以裸盘访问形式直接读写硬盘，从而达到高效的数据访问性能。本文以数据写入(Record Put)为例，来一窥Aerospike的内部逻辑结构(Aerospike参考版本：4.5.3.3)。

<!-- more -->

# 整体流程

一条record(key+value)在Aerospike中的写入过程，大致如下图所示：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/025_aerospike_internal_put_key/img/01_put_procedure.png)

每一条record都有一个指定的key与之关联。最终，server端写入record使用的key并不是原始的user key，而是由set name + user key经由ripemd160算法计算出的20 bytes固定长度的KeyDigest(keyd)

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/025_aerospike_internal_put_key/img/02_record_key_hash.png)

keyd的计算，在client端完成(常规情况下)。而整个record put过程如下：

1. client端基于set name和user key，通过ripemd160算法，计算出keyd

2. client端通过keyd/4096定位该key归属的partition，并查询partition表，将本次put请求发送到对应parition的master节点

3. master节点，为每一个partition构建了一棵广义上的索引tree结构(as_index_tree)。该**<font color=#0000FF>结构实际上为两级索引(hash+rbtree)</font>**。索引步骤为：首先，对keyd取余，找到该keyd对应的子rbtree(sprig)。取余的基数为sprig的配置数量(`artition-tree-sprigs`, 默认值：256)

4. <font color=#0000FF>在找到对应的sprig(rbtree)之后，接着，将对应record key以rbtree节点形式插入到指定位置。</font>

5. 完成索引的构建后，还需要完成record value的写入，**<font color=#FF0000>Aerospike并没有直接将value写入磁盘，而是将value首先写入到swb(`ssd_write_buf`)</font>**，每个swb大小为1M(`write-block-size`)，一个swb将可能写入多条record。Aerospike将ssd磁盘按`write-block-size`为单位切分成若干给write block(wBlock)，swb与唯一的一个wBlock对应(swb记录了wBlockId信息)。在最终将swb落盘时，将swb内容落到对应wBlock的位置即可。

6. 在master节点完成index+record(key+value)的内存写入后，则发送数据到replica server，要求replica server 完成对应的index+record写入

7. replica server根据要求完成index+record写入，并向master节点返回响应。

8. 在**<font color=#FF0000>确认master server + replica server均完成index + record的内存写入后，master节点向client端返回PUT成功的响应</font>**。

9. 当swb被写满（或者是持续一定时间的写入, `flush-max-ms`），swb被加入到`swb_write_q`，准备进行持久化。

10. 后台***<font color=#0000FF>ssd write thread</font>***不断从swb_write_q中拉取swb，将其落盘到ssd device，最终**<font color=#0000FF>完成record(value)持久化</font>**。



# 线程切换逻辑

server在处理record PUT时，主要涉及三类线程处理：
1. service thread: 负责处理io请求
2. transaction thread: 负责将key-value写入内存索引及swb缓存，并将相应返回给调用方
3. write thread: 负责将swb中的数据，异步落盘

<img src="https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/025_aerospike_internal_put_key/img/03_thread_for_record_put.png" width="500" height="" align="center" />


# 调用时序

如下所示，为Aerospike server处理PUT请求的整体调用时序

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/025_aerospike_internal_put_key/img/04_function_call_seq_for_put.png)

* service thread在收到PUT请求后，将PUT transaction通过transaction_queue转交给transaction thread来处理
* transaction thread主要分三步来完成整个处理过程:
 * 通过`write.c::write_master`，完成index+record在master数据索引sprig及磁盘缓存swb上的写入
 * 通过`write.c::start_write_repl_write`，通知replica server完成在副本节点的index+record的内存写入
 * 通过`write.c::write_repl_write_cb`，完成将响应返回给调用方
* transaction thread在write_master进行index+record的内存写入时，进一步的分为2步进行：
 * 通过`record.c::as_record_get_create()`，完成将keyd(index)写入到指定的sprig
 * 通过`storage.c::as_storage_record_write()`，完成将index写入到swb缓存区
* 后台write thread不断的从`swb_write_q`中拉取待持久化的swb，并通过`drv_ssd.c::ssd_flush_swb`将目标swb写入ssd device的指定位置（该位置，在swb被分配的时候，已经确认）




# 数据结构说明

## index索引结构

一个`as_index_tree`对应了一个partition的内存索引结构。如上所述，内存索引结构为一个hash+rbtree的两级结构。如下图所示：

<img src="https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/025_aerospike_internal_put_key/img/05_as_index_tree.png" width="600" height="" align="center" />

一个`as_index_tree`对应了partition的整个索引空间，而一个`as_index_tree`包含了若干个(`partition-tree-sprigs`, 默认256)子树。同时，为控制并发访问子树，而包含了固定的256组`mutex-pair`(一个pair包含2个muext)，用于对子树进行更细力度的访问控制。

每一个`as_sprig`是一个arena内存句柄，通过句柄，可以拿到对应的内存结构

``` cpp
typedef struct as_sprig_s {
    uint64_t n_elements: 24; // max 16M records per sprig
    uint64_t root_h: 40;
} as_sprig;
```

`as_sprig.root_h`对应的是子树的根节点，其内部实际为tree node结构类型：`as_index`

<img src="https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/025_aerospike_internal_put_key/img/06_as_index.png" width="500" height="" align="center" />

`as_index`作为rb-tree node结构，其中**<font color=#0000FF>[tree_id, color, keyd, left_h, righ_h]</font>**部分主要用于维护rb-tree结构，**<font color=#0000FF>[rblock_id, n_rbocks, file_id]</font>**则用于记录该key(index)所对应的value(record)的持久化位置信息，[last_update_time, generation]则用于记录index的metadata信息（ttl相关）

`as_index`通过index.c的若干接口，提供基于rb-tree的insert/get操作
* `as_index_sprig_get_insert_vlock`， 将指定index插入到rb-tree中
* `as_index_sprig_try_get_vlock`， 从rb-tree查找指定index(keyd)

`as_index`中左右子树`left_h`, `righ_h`也不是实际的内存地址，而是基于arena分配的内存地址句柄(间接访问)，在访问所有子树时，均需基于句柄，拿到对应的子节点指针，进而执行访问：

```cpp
#define RESOLVE_H(__h) ((as_index*)cf_arenax_resolve(isprig->arena, __h))

r_h = cmp > 0 ? r->left_h : r->right_h;
r = RESOLVE_H(r_h);
```

在通过keyd查找索引时，首先会基于keyd判断该key归属于哪个sprig: `as_index_sprig_from_keyd(tree, &isprig, keyd);` 这里则是将keyd进行"hash"运算，找到对应的sprig:

```cpp
static inline void
as_index_sprig_from_keyd(as_index_tree *tree, as_index_sprig *isprig,
        const cf_digest *keyd)
{
    // Get the 28 most significant non-pid bits in the digest. Note - this is
    // hardwired around the way we currently extract the (12 bit) partition-ID
    // from the digest.
    uint32_t bits = (((uint32_t)keyd->digest[1] & 0xF0) << 20) |
            ((uint32_t)keyd->digest[2] << 16) |
            ((uint32_t)keyd->digest[3] << 8) |
            (uint32_t)keyd->digest[4];

    uint32_t lock_i = bits >> tree->shared->locks_shift;
    uint32_t sprig_i = bits >> tree->shared->sprigs_shift;

    isprig->destructor = tree->shared->destructor;
    isprig->destructor_udata = tree->shared->destructor_udata;
    isprig->arena = tree->shared->arena;
    isprig->pair = tree_locks(tree) + lock_i;
    isprig->sprig = tree_sprigs(tree) + sprig_i;
    isprig->puddle = tree_puddle_for_sprig(tree, sprig_i);
}
```

在拿到对应的`as_index_sprig`之后，再将key插入到该sprig中：`as_index_sprig_get_insert_vlock`，其中插入逻辑，即为红黑树的节点插入逻辑。


## record缓存结构ssd_write_buffer(swb)

swb用于在内存中记录record(value)内容，其结构`ssd_write_buf`如下所示：

<img src="https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/025_aerospike_internal_put_key/img/07_swb_2.png" width="600" height="" align="center" />

其中，buf用于存储record的实际内容，wblock_id则记录了该swb对应的持久化ssd上的`wblock_id`编号，后续对swb的持久化，就是将swb内容写入到对应的wblock中

## record记录结构as_flat_record

swb->buf存储若干个record(value)，每个record的内容则已`as_flat_record`形式组织，并存放在swb->buf中:

<img src="https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/025_aerospike_internal_put_key/img/08_flat_record.png" width="400" height="" align="center" />

头部是若干record的meta信息，紧跟其后的，则是实际的colume(bin),每一个bin都有自己的name以及实际的数据，根据bin的类型，可能存放的是integer bin(integer_flat), 可能是list bin(list_flat) ... 这部分record内容，既是内存中的组织形式，同样也是最终持久化到ssd device的组织形式。




# 总结

从上述对Aerospike record put整个过程的描述，我们可以总结出：

1. 针对每个partition， 以hash+rb-tree方式进行索引组织(`as_index_tree`)
2. 第1级访问，是对keyd进行"hash"再取模，hash过程是对keyd的位移操作(`as_index_sprig_from_keyd`)
3. 第2级rb-tree(as_index)的组织形式为标准红黑树访问方式
4. **<font color=#FF0000>Aerospike的数据持久化是异步落盘</font>**
5. 落盘以1M的swb->buf为单位(`write-block-size`配置)
6. 每一行(record)以`as_flat_record`方式组织，持久化到ssd device
7. **<font color=#FF0000>当master node确认index+record写入内存，且replica node也已写入内存即向client返回响应</font>**


