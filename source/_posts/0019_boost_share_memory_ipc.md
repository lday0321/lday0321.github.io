---
title: 共享内存(ipc)的使用
date: 2017-12-17 23:08:14
tags: 
	- cpp
	- boost
	- share memory
	- ipc
---

共享内存，是进程间数据传输的方式之一。数据发送方将数据放入共享内存中，数据接收方则从共享内存中将数据读出，进而完成整个数据的传输。这里我通过使用场景的方式简单总结`boost::interprocess share_memory`的使用


<!-- more -->

# 概述

在Linux平台下，操作系统为用户提供了两套共享内存的接口：

| System-V共享内存接口       |描述|
| -------- | ----- |
| int shmget(…) | 创建一块共享内存 |
| int shmat(…) | 附接一块共享内存 |
| int shmdt(…) | 断接一块共享内存 |
| int shmctl(…) | 获取共享内存对象信息，删除一块共享内存 |

| Posix共享内存接口      |描述|
| -------- | ----- |
| int shm_open(…) | 创建or打开共享内存对象 |
| int ftruncate(…) | 设置共享内存的大小 |
| int fstat(…) | 获取共享内存对象信息 |
| int shm_unlink(…) | 删除共享内存 |
| void *mmap(…) | 将共享内存对象映射到本地进程空间中 |

在Windows平台下，操作系统为用户提供的共享内存接口为：

| Windows平台共享内存接口      |描述|
| -------- | ----- |
| HANDLE CreateFileMapping(…) | 创建一块共享内存 |
| HANDLE OpenFileMapping(…) | 附接一块共享内存 |
| void *MapViewOfFile(…) | 将共享内存映射到本地进程空间中 |
| void UnmapViewOfFile(…) | 释放共享内存 |
| void CloseHandle(…) | 关闭共享内存映射文件句柄 |

不同平台提供了不同的共享内存使用接口。C++ Boost库对两类平台的共享内存接口和使用方式提供了进一步抽象和封装，并将该部分功能放置在interprocess库中。完整的interprocess库介绍可参考：http://www.boost.org/doc/libs/1_57_0/doc/html/interprocess.html

## 场景1：共享内存池的创建及销毁

在使用共享内存之前，我们都需要创建一块共享内存。直接使用操作系统的共享内存，则在不同的平台调用指定接口即可。`Boost::ipc`，则需要首先创建一个托管共享内存池，所有的共享内存对象，都基于该内存池来分配内存。

本文描述的使用场景主要依赖下述头文件：

```cpp
#include <boost/interprocess/managed_shared_memory.hpp>
#include <boost/interprocess/shared_memory_object.hpp>
#include <boost/interprocess/allocators/allocator.hpp>

namespace bip = boost::interprocess;
```

共享内存池的创建：

```cpp
static char *shareMemoryBlockName = "MySharedMemory";
static int32_t sharedMemoryBlockTotalSize = 65536;  

//创建共享内存
bip::managed_shared_memory segment(bip::create_only,
	shareMemoryBlockName,  //segment name
	sharedMemoryBlockTotalSize);
```

共享内存的附接：

```cpp
//首先attach共享内存
//首先需要attach到share memory block
bip::managed_shared_memory segment(bip::open_only,
	shareMemoryBlockName  //segment name
	);
```

对应的，根据共享内存池名称，可以将共享内存池删除：

```cpp
//同时销毁共享内存
bool isRemoved = bip::shared_memory_object::remove(shareMemoryBlockName);
if (isRemoved) {
	cout << "Share Memory=[" << shareMemoryBlockName << "] removed!" << endl;
}
else {
	cout << "Share Memory=[" << shareMemoryBlockName << "] not removed!" << endl;
}
```

<font color="#FF0000">【注】：使用时需确保segemnt对象整个生命周期的完整性，不能在segement已经析构后，还使用由该共享内存池创建的对象。</font>

## 场景2：共享内存对象的创建及销毁

在共享内存池创建好之后，则可以基于共享内存池创建对象。在共享内存池上可创建有名/无名对象：

### 1. 有名对象的创建、附接、删除

有名对象的创建时，需要给该对象分配指定的对象名称：

```cpp
//bip::managed_shared_memory segment

//有名对象的创建
MyType *instance = segment.construct<MyType>  
         ("MyType instance")  //name of the object  
         (0.0, 0);            //ctor first argument 
```

创建有名共享内存对象的好处是，其他进程可以通过该对象名称，基于同一个共享内存池，方便的定位到该对象。对应的需要附接到该共享内存对象，则只需通过共享内存池找到指定命名对象：

```cpp
//bip::managed_shared_memory segment

//有名对象的附接
typedef std::pair<MyType *, bip::managed_shared_memory::size_type> ResultT;
ResultT res = segment.find<MyType>("MyType instance");
if (!res.second)
{
	cout << "could not find MyType instance" << endl;
}
```

有名对象的删除，则通过调用destroy接口完成删除动作：

```cpp
//bip::managed_shared_memory segment

//有名对象的删除
segment.destroy<MyType>("MyType instance");
```

### 2. 匿名对象的创建、使用及销毁

#### 2.1 基于偏移指针的共享数据使用

大多数时候，我们其实不打算使用有名对象，而是使用无名对象，此时，我们需要使用共享内存池提供的匿名对象执行共享内存对象的创建：

```cpp
//bip::managed_shared_memory segment

//匿名对象的创建
MyType *ptr = segment.construct<MyType>(bip::anonymous_instance) (par1, par2...);  
```

由于匿名对象无法通过名称指定定位，因此，当我们将一个匿名对象由进程A写入共享内存时，进程B没法直接拿到共享内存，进而使用。Boost::interprocess提供了方法来使得进程B能够获取匿名对象的内容。具体做法是：使用boost::interprocess::offset_ptr。我们在共享内存中创建匿名共享内存对象，通过offset_ptr指向共享内存对象。将offset_ptr由进程A发送到进程B，进程B再通过offset_ptr将实际内容读出来。由于offset_ptr指向的对象是存放在共享内存中，因此，对应的对象应当具备从共享内存中分配内存空间的能力，也就是说该对象的内存分配器应当指向对应共享内存的内存分配器。

```cpp
#include <boost/interprocess/managed_shared_memory.hpp>
#include <boost/interprocess/allocators/allocator.hpp>
#include <boost/interprocess/containers/string.hpp>

namespace bip = boost::interprocess;   //给ipc设置别名

//首先定义共享内存分配器类型
typedef bip::allocator<char, bip::managed_shared_memory::segment_manager> CharAllocator;
typedef bip::basic_string<char, std::char_traits<char>, CharAllocator> shared_string;

struct IPCData
{
	shared_string name;
	int       id;
	boost::atomic<int> atomicShrMmOrderRefCnt;

	IPCData(const shared_string::allocator_type& al):name(al),atomicShrMmOrderRefCnt(1)
	{
		id = 0;
	}

};

//定义数据的OffsetPtr
typedef bip::offset_ptr<IPCData> IPCDataOffsetPtrT;

//定义该offsetPtr的内存分配器
typedef bip::allocator<IPCDataOffsetPtrT, bip::managed_shared_memory::segment_manager>IPCDataOffsetPtrAllocatorT;
```

定义好对应的偏移地址指针后，就可以通过创建匿名共享内存对象的方式，将共享内存对象托管给偏移地址指针：

```cpp
//bip::managed_shared_memory segment

//创建匿名共享内存offset_ptr指针
IPCDataOffsetPtrT ipcDataOffsetPtr =
	segment.construct<IPCData>(bip::anonymous_instance)
		(shared_string::allocator_type(segment.get_segment_manager()));
```

此后，定义相应的共享内存通道:

```cpp
#include <boost/interprocess/containers/list.hpp>

namespace bip = boost::interprocess;   //给ipc设置别名

//定义基于该offsetPtr以及对应的内存分配器的queue
typedef bip::list<IPCDataOffsetPtrT, IPCDataOffsetPtrAllocatorT> IPCDataOffsetPtrQueueT;
```

让进程A将offset_ptr放入通道：

```cpp
//bip::managed_shared_memory segment
//ipcDataOffsetPtr 

//找到对应的allocator
const IPCDataOffsetPtrAllocatorT alloc_inst(segment.get_segment_manager());

IPCDataOffsetPtrQueueT *pIPCDataOffsetPtrQueue;
pIPCDataOffsetPtrQueue =
	segment.construct<IPCDataOffsetPtrQueueT>
		(channelName.c_str())(alloc_inst);


pIPCDataOffsetPtrQueue->push_back(ipcDataOffsetPtr );
```

并由进程B从通道中读取数据：

```cpp
//bip::managed_shared_memory segment
//ipcDataOffsetPtr 

ipcDataOffsetPtr = pIPCDataOffsetPtrQueue->front();
IPCDataOffsetPtrQueue->pop_front();

cout <<ipcDataOffsetPtr->name << endl;

//此后ipcData需要通过destroy_ptr进行内存释放
segment.destroy_ptr(ipcDataOffsetPtr->get());
```

使用完的匿名共享内存对象，则通过共享内存池的destroy_ptr完成内存释放。

#### 2.2 基于内存句柄的共享数据使用

上述匿名对象的使用需要通过offset_ptr作为跨越进程的桥梁，在两个进程间传输offset_ptr，之后再通过offset_ptr间接的获取到对应的共享内存对象。该方法能够达到数据共享的目的，但有两点不足：

1. 该方法似乎有点绕，不够直观; 
2. 改方法由于传递的是偏移指针，该指针并非朴素类型(POD)，无法使用boost的无所队列进行传递。

是否有更简单的方法，使得我们可以有更加方便的方式在进程间共享数据呢？

答案是：可以通过`boost::interprocess::managed_shared_memory::handle`。通过`handle`，我们可以方便的还原出该`handle`对应对象的指针，同时，通过指针我们也能够方便的获得该指针对应的`handle`。重要的是，而实际上，`handle`就是朴素类型的`int`, 我们能够方便的将其通过无锁队列的`Queue`由一个进程推送到另外一个进程，进而达到在进程间高效共享数据的目的。下面我们来看看基于`handle`+`lockfree::queue`来完成进程间数据传输。

##### 1. 简单定义我们需要在进程间共享的数据结构

该数据类型包含3个字段：id、id64和name

```cpp
#ifndef __IPC_HANDLE_DATA_H__
#define __IPC_HANDLE_DATA_H__

#include <boost/interprocess/managed_shared_memory.hpp>
#include <boost/interprocess/allocators/allocator.hpp>
#include <boost/interprocess/containers/string.hpp>

namespace bip = boost::interprocess;   //给ipc设置别名

//首先定义共享内存分配器类型
typedef bip::allocator<char, bip::managed_shared_memory::segment_manager> CharAllocator;
typedef bip::basic_string<char, std::char_traits<char>, CharAllocator> shared_string;


//用于IPCHandle传输
class IPCHandleData {
public:
	IPCHandleData(const shared_string::allocator_type& al);
	~IPCHandleData();

	int id;
	__int64 id64;
	shared_string name;

};
#endif
```

##### 2. 定义我们需要用到的共享内存句柄类型

以及共享内存中定义无锁队列queue需要的分配类型

```cpp
/定义共享内存handle类型
typedef bip::managed_shared_memory::handle_t IPCHandleT;

//定义该handle的共享内存分配器
typedef bip::allocator<IPCHandleT, 
			bip::managed_shared_memory::segment_manager> 
								MyIPCHandleAllocatorT;

//定义基于该IPCHandle以及对应的内存分配器的Queue
typedef boost::lockfree::queue<IPCHandleT, 
			boost::lockfree::capacity<65534>,  //最多时2^16-2
			boost::lockfree::allocator<MyIPCHandleAllocatorT>> 
								IPCHandleMwMrQueueT;
```

##### 3. 由一个创建者(Creator)负责创建共享内存

并创建用于传输的无锁队列

```cpp
void Creator()
{
	cout << "Creator" << endl;

	// 删除遗留的共享内存
	bip::shared_memory_object::remove("MySharedMemory");

	// 构建新的托管共享内存区
	bip::managed_shared_memory segment(bip::create_only,
		"MySharedMemory",  //segment name
		65536 * 1024 * 10);

	// 定义托管共享内存区的分配器
	const MyIPCHandleAllocatorT alloc_inst(segment.get_segment_manager());

	try
	{
	
	// 创建共享内存中的用于传递IPCHandleT的无锁队列
	IPCHandleMwMrQueueT *pMyIPCHandleQueue =
		segment.construct<IPCHandleMwMrQueueT>("MyIPCHandleQueue")(alloc_inst);
	}
	catch (exception &e)
	{
		cout << e.what() << endl;
	}

	cout << "Creator is waiting here..." << endl;
	system("pause");

	
}
```

##### 4. 生产者一方首先附接到共享内存

进一步找到对应的无锁队列。基于共享内存，创建待传输的共享内存对象，并找到该共享内存对象的句柄handle，将该句柄handle放入到无锁队列中，完成传输动作

```cpp
void Producer()
{
	cout << "Producer" << endl;

	//附接目标共享内存
	bip::managed_shared_memory segment(bip::open_only,
				"MySharedMemory");  //segment name
											
	typedef std::pair<IPCHandleMwMrQueueT *, 
			bip::managed_shared_memory::size_type> ResultT;
	ResultT res = segment.find<IPCHandleMwMrQueueT>("MyIPCHandleQueue");

	cout << "res.second=" << res.second << endl;
	if (!res.second)
	{
		cout << "could not find \"MyIPCHandleQueue\"" << endl;
		return;
	}
	else
	{
		cout << "find \"MyIPCHandleQueue\"" << endl;
	}

	IPCHandleMwMrQueueT *pMyIPCHandleQueue = res.first;

	int id = 1;
	
	//创建匿名共享内存对象，销毁时需要用destroy_ptr
	IPCHandleData *pIPCHandleData =
 		segment.construct<IPCHandleData>
			(bip::anonymous_instance)
				(shared_string::allocator_type(segment.get_segment_manager()));
	pIPCHandleData->id = id;
	pIPCHandleData->id64 = id;

	stringstream ss;
	ss << "name-" << id;
	pIPCHandleData->name = ss.str().c_str();

	cout << "cur IPCHandleData :" << endl;
	cout << "id=" << pIPCHandleData->id << endl;
	cout << "id64=" << pIPCHandleData->id64 << endl;
	cout << "name=" << pIPCHandleData->name << endl;

	//获取共享内存对应的handle
	IPCHandleT curHandle = segment.get_handle_from_address(pIPCHandleData);
	cout << "handle=" << (int)(curHandle) << endl;

	//将该handle放入queue中，传递给对端
	pMyIPCHandleQueue->push(curHandle);

}
```

##### 5. 消费者同样附接到共享内存

并找到对应的Queue，从Queue中pop出handle数据，基于该handle，找到对应的共享内存指针，进而获取共享内存对象的信息

```cpp
void Consumer()
{
	cout << "Consumer" << endl;

	//附接目标共享内存
	bip::managed_shared_memory segment(bip::open_only,
		"MySharedMemory");  //segment name
		
	typedef std::pair<IPCHandleMwMrQueueT *, 
				bip::managed_shared_memory::size_type> ResultT;
	ResultT res = segment.find<IPCHandleMwMrQueueT>("MyIPCHandleQueue");

	cout << "res.second=" << res.second << endl;
	if (!res.second)
	{
		cout << "could not find \"MyIPCHandleQueue\"" << endl;
		return;
	}
	else
	{
		cout << "find \"MyIPCHandleQueue\"" << endl;
	}

	IPCHandleMwMrQueueT *pMyIPCHandleQueue = res.first;

	IPCHandleT recvHandle;
	if (pMyIPCHandleQueue->pop(recvHandle))
	{
		cout << "Recv data" << endl;

		//接收到消息后，通过handle找到对应的共享内存对象
		IPCHandleData *pIPCHandleData = 
			(IPCHandleData *)segment.get_address_from_handle(recvHandle);

		//输出共享内存对象信息
		cout << "recved IPCHandleData :" << endl;
		cout << "id=" << pIPCHandleData->id << endl;
		cout << "id64=" << pIPCHandleData->id64 << endl;
		cout << "name=" << pIPCHandleData->name << endl;

		//最终释放该共享内存对象
		segment.destroy_ptr(pIPCHandleData);
	}
}
```

基于handler，最终我们可以在两个不同的进程中访问同一片共享内存数据。

