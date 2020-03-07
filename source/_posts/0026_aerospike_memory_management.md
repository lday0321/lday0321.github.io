---
title: Aerospike的内存管理
date: 2020-03-07 21:56:14
tags: 
	- Aerospike
	- NoSQL
	- Memory
	- Jemalloc
---

[Aerospike](https://github.com/aerospike)使用[jemalloc](http://jemalloc.net)进行内存分配和管理。利用了[jemalloc](http://jemalloc.net)提供的扩展特性来更细力度的进行内存控制。

<!-- more -->

# jemalloc的扩展能力

1. jemalloc提供扩展接口来创建独立内存管理单元`arena`。可基于该`arena`进行内存申请和释放，arena之间的内存管理无任何冲突。
 * `mallctl("arenas.extend", &arena, &arena_len, NULL, 0)`，用于创建`arena`。
 * `mallocx(size, arena_flags)`用于在指定`arena`上申请内存空间
 * `free(p)`释放接口与标准接口无差。

2. jemalloc提供扩展接口来创建线程级别的cache(`tcache`)，分配内存空间时，可要求优先使用`tcache`进行空间分配(`tcache`空间优先），使用`tcache`的好处是，完全无冲突。因为一个`arena`还是有可能被分享给不同的线程共享使用，还是存在共享冲突的可能。
 * `mallctl("tcache.create", &tcache, &len, NULL, 0)`，用于创建`tcache`。

3. jemalloc提供内存状态信息输出，方便查看定位内存使用情况：
 * `mallctl("stats.allocated", &allocated, &len, NULL, 0)`，获取分配的内存数量
 * `mallctl("stats.active", &active, &len, NULL, 0)`，获取分配的active pages信息

# Aerospike如何使用jemalloc

下图为Aerospike使用jemalloc的大致关系：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0026_aerospike_memory_management/img/01_aerospike_use_jemalloc.png)

1. Aerospike启动时，创建了150个`arena`(`#define N_ARENAS 150`)，线程在使用jemalloc进行内存分配时，会在第一次调用时，以round-robin的方式attach到其中一个`arena`上，后续的内存管理动作均基于该`arena`完成。
 ![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0026_aerospike_memory_management/img/02_aerospike_alloc_arena.png)

2. Aerospike为每一个对应的`namespace`创建了一个指定的`arena`。当`namespace`指定了`data-in-memory`参数时(record数据将在内存中保留一份)，此时该`namespace`的record内容，都将关联到该arena上来完成内存分配。
 ![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0026_aerospike_memory_management/img/03_aerospike_namespace_arena.png)



在具有一个`namespace`的实验环境下，我们可以看到，server总共分配出151(150+1)个`arena`，通过`mallctl("arenas.narenas",...)`进行统计，结果如下：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0026_aerospike_memory_management/img/04_aerospike_jemalloc_arena_count.png)

## 使用jemalloc的心得

之所以Aerospike以上述形式进行内存管理，是根据Aerospike研发自身的经验总结：
1. 对于大多数在内存处理任务来说，per thread `arena`已经非常好了，因为大多数内存会在当前线程中申请，并在当前线程中释放( *<font color=#0000FF>This works well with a purely-transactional model where the data is allocated and relatively quickly freed on the same thread.</font>* )

2. 但是对于数据存储而言，由于数据访问的存储周期会跨多个线程，基于`namespace`的内存管理(以`namespace`为单位来构建`arena`，并基于该`arena`来管理内存）会更加有效( *<font color=#0000FF>The main database object store, however, follows entirely different allocation patterns. These objects are usually persistent, potentially much larger, and may be accessed over time by many different threads (and sometimes are even accessed concurrently.) Therefore, the best pattern we have found is to have objects with the same general characteristics (i.e., those within the same Aerospike namespace, which roughly corresponds to a database</font>* )

Aerospike在使用jemalloc的总结可参考：[In-Memory Computing At Aerospike Scale: When To Choose And How To Effectively Use JEMalloc](http://highscalability.com/blog/2015/3/17/in-memory-computing-at-aerospike-scale-when-to-choose-and-ho.html)。从文章数据来看，Aerospike改用jemalloc之后，内存使用效率变高，更加稳定：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0026_aerospike_memory_management/img/05_aerospike_jemalloc_result.png)

> <font color=#0000FF>After selecting a better dynamic memory allocator and changing the server to use the allocator effectively in a multi-threaded context, the system memory use per node quickly “flatlines” on the right side (with some residual “bouncing” as memory as additional memory is transiently allocated and given back to the system.) </font>

同时，Aerospike还利用了jemalloc扩展接口，来统计进程的内容使用情况`alloc.c::cf_alloc_heap_stats()`:
* `mallctl("stats.allocated", ...)`
* `mallctl("stats.active",...)`
* `mallctl("stats.mapped",...)`

Aerospike后台会定时输出该类信息：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0026_aerospike_memory_management/img/06_aerospike_memory_info_output.png)

通过asinfo命令也可以直接获取该信息:

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0026_aerospike_memory_management/img/07_aerospike_asinfo_memory_info.png)


# Aerospike自身的内存管理结构arenax

上面我们提到，Aerospike会为每个`namespace`创建一个对应的`arena`。在设置了`data-in-memory`时，这个`arena`用于管理该`namespace`的所有record的内存。请注意，这个`arena`**只负责管理record部分，也就是value部分的内存**。而无论是否设置`data-in-memory`，用于组织key的`sprig`(红黑树索引结构），并没有使用jemalloc提供的arena来管理。由于`sprig`的树节点(`as_index`)结构为定长的64byte，Aerospike使用了自己设计的`arenax`结构来管理该定长数据。

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0026_aerospike_memory_management/img/08_aerospike_spig_arenax.png)

上图为`arenax`的组织结构。`arenax`为两层内存分配方式结构：

* 第一层为`stage`，总共有256个`stage`，每个`stage`指向一块定长的内存区域（1G)。
* 第二层，则为`stage`指向的这片内存区域，该片区域根据`as_index`的大小被切分成1677721块(1G/64)。
* `arenax`维护了一个`free list`。当有`as_index`结构free时，则放回到该`free list`的头部。`arenax`实际维护的是该`free list`的头。
* 当有malloc请求时，arenax首先会查看free list里面是否还有空闲slot，如果有的话，则将free list头上的slot返回给调用方。
* 如果`free list`没有空闲slot，则从根据当前`stage_id`（at_stage_id）以及对应的`element_id`(at_element_id)拿到可用的slot，将该slot返回给调用方，并顺势调整at_stage_id, at_element_id。
* 返回给调用方的是`cf_arenax_handle`（uint64_t)，该句柄并不是内存地址，而是一个两级索引，**<font color=#0000FF>前36位为第1级stage索引，后28位为第2级elemnt索引</font>**。在使用时，需要通过handle调用接口转换出指定的内存指针: `void* cf_arenax_resolve(cf_arenax* arena, cf_arenax_handle h)`。
* 光从`arenax`的组织来看，单台server，一个`namespace`的索引节点上限`(as_index)`数量：`stage_count * slot_count_per_stage = 256*16777216 = 4294967296`。基本上算无上限。
* 为区分非法`handle`，`stage[0]`的`slot[0]`不做分配，初始状态是`at_stage_id = 0, at_element_id = 1`。

