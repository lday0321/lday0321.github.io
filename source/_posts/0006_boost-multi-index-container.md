---
title: 多维索引容器(multi_index_container)的使用
date: 2017-03-19 13:34:18
tags:
	- cpp
	- boost
---

C++ STL库为我们提供了vector/list/queue/(unordered)set/(unordered)map等各类容器，这些容器各自有各自的特点。有的提供链表类型的连续访问(vector/list/queue)，有的提供平衡二叉树的数据组织结构(set/map)，有的提供基于Hash的随即定位访问(unordered_set/unordered_map)。但有些时候，单一类型的访问并无法满足我们的需求，例如，有这样一个需求，需要我们实现一个LRU Cache。LRU的意思是：Least Recently Used，即最近最久未被使用的意思。LRU Cache的意思是：如果一个数据在最近一段时间没有被访问到，那么在将来它被访问的可能性也很小，基于这个原则，我们希望该类Cache的空间已经存满数据时，应当把最久没有被访问到的数据淘汰。

<!-- more -->

理解了LRU Cache的含义后，我们进一步思考，我们应当用什么样的数据结构来管理整个LRU Cache呢？首先，第一个问题，用什么样的数据结构管理数据，能够体现最近最久未被使用这层含义呢？使用double linked List(STL中的deque提供类似的功能)，每次有新数据被访问，就将该数据放在deque的头部，如果此时cache空间已满，则需要首先移出deque尾部数据。deque越靠经头部的数据越是recently used(越最近被用过)。是不是deque就足够了？显然不是，通过deque我们只处理了“新”数据被访问的情况。我们还需要处理原本就在Cache中的“老数据”再度被访问的场景，需要调整该数据在deque中的位置，将其重新放置到deque头部。这个问题看似挺好解决，分两步执行操作：首先将这个“老数据”从deque中移除，然后再将这个数据加入到deque的头部即可了。将数据重新加入到deque头部是相对简单的，但如果才能迅速的找到“老数据”，并将他从原来的位置移除，这个操作看似简单，却并不简单。如果需要从deque中找到这个“老数据”，我们必须得遍历整个deque，因此从deque中删除“老数据”的时间复杂度将会是O(n)。作为一个Cache，每次数据命中之后都必须执行一个O(n)的调整操作，显然代价太大了。那能否有别的办法呢？最直观的想法就是用过hash直接能找到“老数据”，找到之后将其从deque上移除，并重新插入到deque头部。这里，实际上我们就是在构建一个2维索引：deque+hash。

![](http://og43lpuu1.bkt.clouddn.com/multi_index/png/0001_multi_index_01.png)

在第0维Deque索引上，维护的是按最近被访问数据排列的数据顺序。而在第1维Hash索引上，则直接维护的1:1的Key(memory Address)—value索引信息，用于在需要进行查找操作时，直接通过key定位到所需的cache。当然，目前为止，我们还仅仅给出了LRU Cache的基本组织结果，为满足LRU Cache的整体功能，我们还需要实现在某一维索引上执行插入(`deque::push_back`, `deque::push_front`, `hash::insert`)操作时，对另外一维索引的同步调整，以及在某一维索引上执行删除操作时，同样的在另外一维索引上的同步调整。整个实现不能说特别复杂，但至少还是需要一定的工作量，以及周全的思考的。

有没有什么工具能够帮我们来实现这样一个多维索引结构呢？STL中没有，但Boost中有。这个结构就是今天要介绍的多维索引容器：multi_index_container。多维索引结构就是如上图LRU Cache结构类似的结构，他为用户提供了统一的多维索引视图，允许用户定义一个多维索引容器上的每一维索引，并为每维索引提供访问接口，为用户屏蔽了在进行插入、删除以及更新时的复杂的多维索引间同步调整逻辑，使得用户在定义好多维索引容器之后就能够非常方便的使用多维索引结构。

要使用boost的multi_index_container，简单来说分为两步：1. 定义多维索引容器；2.使用多维索引容器。

**定义多维索引容器**
多维索引容器定义时，需要指定：1. 容器中存放的元素类型；2. 容器上构建多少维索引，各维索引如何组织等信息。

定义多维索引容器的难点主要在于：2.定义容器上构建的索引。
    
并不是什么类型的索引都能够作为`boost::multi_index_container`的索引。boost为multi_index_container提供了5类索引类型：

| Boost.MutiIndex类型        | 对应类比的STL容器类型   | 
| :--------   | :-----  | 
| ordered_unique     | set |
| ordered_non_unique | multiset |
| hashed_unique      | unordered_set |
| hashed_non_unique  | unordered_multiset |
| sequenced          | list (deque) |

类比我们对LRU Cache的设计，我们可以选择multi_index_container中的sequenced + hashed_unique两个索引。对于使用`stl::set/unordered_set`时，我们自然知道，针对实例化的元素类型，我们需要为stl::set提供ElementLessComp调用对象，为unordered_set提供ElementEqualComp和ElementHashValue两个调用对象。在multi_index_container中定义对应的hashed_unique索引类型时，也必须给出对应的两个可调用对象。
    
假设我们的LRU Cache中存放的是智能指针对象，且在deque和hash之外再多构建一维索引：set，那么我们将可以按如下形式来定义一个多维索引容器：
    
依赖的头文件:

```cpp
#include <boost/multi_index_container.hpp>
#include <boost/multi_index/ordered_index.hpp>	     //有序索引，对应(ordered)set, 包括multiset
#include <boost/multi_index/hashed_index.hpp>	     //散列索引，无序，对应(unordered)hash
#include <boost/multi_index/sequenced_index.hpp>     //序列索引，list
#include <boost/multi_index/key_extractors.hpp>	     //键提取器
	
using namespace boost::multi_index;
```
    
容器元素定义：
    
```cpp
class MyCustomizedClass10SmartPtrIdLessComp;
class MyCustomizedClass10SmartPtrWholeEqualComp;
class MyCustomizedClass10SmartPtrWholeHash;
    
class MyCustomizedClass10
{
public:

	MyCustomizedClass10(int id, const string &f, const string &l):
		m_id(id), m_fname(f), m_lname(l)
	{}

	const string & first_name() const 			//const成员函数，取m_fname
	{
		return m_fname;
	}

	string & last_name() 					//非const成员函数，取m_lname
	{
		return m_lname;
	}

	friend ostream & operator << (ostream & out, const MyCustomizedClass10 & c);
	friend class MyCustomizedClass10SmartPtrIdLessComp;
	friend class MyCustomizedClass10SmartPtrWholeEqualComp;
	friend class MyCustomizedClass10SmartPtrWholeHash;


private:
	int m_id;							//身份证id
	string m_fname, m_lname;			//姓名
};

typedef boost::shared_ptr<MyCustomizedClass10> MyCustomizedClass10Ptr;
    ```

用于定义索引的可调用对象定义：
```cpp
class MyCustomizedClass10SmartPtrIdLessComp
{
public:
	bool operator()(const MyCustomizedClass10Ptr &lhs, 
				const MyCustomizedClass10Ptr &rhs)
	{
		cout << "MyCustomizedClass10SmartPtrIdLessComp call" << endl;
		return lhs->m_id < rhs->m_id;
	}
};

class MyCustomizedClass10SmartPtrWholeEqualComp
{
public:
	bool operator()(const MyCustomizedClass10Ptr &lhs, 
				const MyCustomizedClass10Ptr &rhs) const //const不能少！
	{
		cout << "MyCustomizedClass10SmartPtrWholeEqualComp call" << endl;
		return lhs->m_id == rhs->m_id &&
				lhs->m_fname == rhs->m_fname &&
				lhs->m_lname == rhs->m_lname;
	}
};


class MyCustomizedClass10SmartPtrWholeHash
{
public:
	std::size_t operator()(const MyCustomizedClass10Ptr lhs) const
	{
		std::size_t seed = 0u;

		boost::hash_combine(seed, boost::hash_value(lhs->m_id));
		boost::hash_combine(seed, boost::hash_value(lhs->m_fname));
		boost::hash_combine(seed, boost::hash_value(lhs->m_lname));

		cout << "MyCustomizedClass10SmartPtrWholeHash seed:" << seed << endl;
		return seed;
	}
};
```

对多维索引容器需要的各位索引类型进行定义：
```cpp
// 1.直接针对整个对象做序列索引,list
typedef sequenced<> c10_idx_0_seq;

// 2.针对id做有序单键索引，set								
typedef ordered_unique<identity<MyCustomizedClass10Ptr>,
				MyCustomizedClass10SmartPtrIdLessComp> c10_idx_1_set;

// 3.针对name做散列索引 hash
typedef hashed_unique<identity<MyCustomizedClass10Ptr>, 
			MyCustomizedClass10SmartPtrWholeHash,
			MyCustomizedClass10SmartPtrWholeEqualComp> c10_idx_2_hash;		
```

最后，基于定义的索引类型，给出最终多维索引容器的定义
```cpp
//定义一个容纳MyCustomizedClass1的多维索引容器
typedef multi_index_container<
		MyCustomizedClass10Ptr,	//容纳的对象是智能指针类型
		indexed_by<			//设置多维索引
			c10_idx_0_seq,
			c10_idx_1_set,
			c10_idx_2_hash
		> 	//索引定义结束
> mic_c10_t;	//最终的多维索引类型
```

完成上述定义滞后，一个完整的多维索引容器类型mic_c10_t就已经定义好了，后续则可以根据定义实际的容器对象，并执行相应的操作。对于多维索引容器的操作，实际上是基于多维索引容器上的索引执行相应的操作。在执行增、删、改时，我们实际上是通过索引进行操作。

```cpp
static int Case07_MultiIndexContainerSmartPointerElement()
{
	MyCustomizedClass10Ptr pMyCC10_0 = boost::make_shared<MyCustomizedClass10>
										(2, "agent", "smith");
	MyCustomizedClass10Ptr pMyCC10_1 = boost::make_shared<MyCustomizedClass10>
										(20, "lee", "someone");
	MyCustomizedClass10Ptr pMyCC10_2 = boost::make_shared<MyCustomizedClass10>
										(1, "anderson", "neo");
	MyCustomizedClass10Ptr pMyCC10_3 = boost::make_shared<MyCustomizedClass10>
										(10, "lee", "bruce");


	mic_c10_t mic;
	mic_c10_t::nth_index<0>::type & listIdx = mic.get<0>();
	mic_c10_t::nth_index<1>::type & setIdx = mic.get<1>();
	mic_c10_t::nth_index<2>::type & hashIdx = mic.get<2>();

	listIdx.push_back(pMyCC10_0);
	listIdx.push_back(pMyCC10_1);
	setIdx.insert(pMyCC10_2);
	hashIdx.insert(pMyCC10_3);

	//通过hash查找元素
	mic_c10_t::nth_index<2>::type::iterator itr =
			hashIdx.find(pMyCC10_2);
	if(itr != hashIdx.end())
	{
		cout << "Find pMyCC10_2 by Hash Idx" << endl;
		cout << **itr << endl;
	}

	//dup
	cout << "Duplication" <<endl;
	std::pair<mic_c10_t::nth_index<0>::type::iterator, bool> result =
			listIdx.push_back(pMyCC10_0);
	cout << "Dup Push Result : " << result.second;

	TraverOutput4On0thIdx(mic);
	TraverOutput4On1stIdx(mic);
	TraverOutput4On2ndIdx(mic);

	return 0;
}
```

通过不同索引对容器数据进行遍历
```cpp
ostream & operator << (ostream & out, const MyCustomizedClass10 & c)
{
	out << "[Id:" << c.m_id << ", name:" << c.m_lname << " " << c.m_fname << "]";
	return out;
}


void TraverOutput4On0thIdx(mic_c10_t mic)
{
	mic_c10_t::nth_index<0>::type & listIdx = mic.get<0>();

	cout << endl;
	cout << "0th Index(list):" << endl;
	mic_c10_t::nth_index<0>::type::iterator listIdxItr =
			listIdx.begin();
	for( ; listIdxItr != listIdx.end(); ++listIdxItr)
	{
  		//list索引返回的对象是不可被修改的！
		const MyCustomizedClass10Ptr pMC = *listIdxItr;  
		cout << *pMC  << endl;
	}
	cout << endl;
}


void TraverOutput4On1stIdx(mic_c10_t mic)
{
	mic_c10_t::nth_index<1>::type & setIdx = mic.get<1>();

	cout << endl;
	cout << "1st Index(set):" << endl;
	mic_c10_t::nth_index<1>::type::iterator setIdxItr =
			setIdx.begin();
	for( ; setIdxItr != setIdx.end(); ++setIdxItr)
	{
		const MyCustomizedClass10Ptr pMC = *setIdxItr;
		cout << *pMC << endl;
	}
	cout << endl;
}


void TraverOutput4On2ndIdx(mic_c10_t mic)
{
	mic_c10_t::nth_index<2>::type & hashIdx = mic.get<2>();

	cout << endl;
	cout << "2nd Index(hash):" << endl;
	mic_c10_t::nth_index<2>::type::iterator hashIdxItr =
			hashIdx.begin();
	for( ; hashIdxItr != hashIdx.end(); ++hashIdxItr)
	{
		const MyCustomizedClass10Ptr pMC = *hashIdxItr;
		cout << *pMC << endl;
	}
	cout << endl;
}
```

当然通过多维索引容器除了执行插入、删除操作外，还可能需要调整key值等修改操作。和不同容器的迭代器不同，多维索引容器获取的迭代器为常量类型，不允许随便修改，这是因为多维索引容器的修改涉及到各维索引及位置的“联动”。多维索引容器提供了modify方法用于修改索引key值，进一步的对于多维索引的详细操作使用说明可以参考[boost的官方文档][1]。

[1]: http://www.boost.org/doc/libs/1_57_0/libs/multi_index/doc/index.html