---
title: 细看Go中的切片(slice)
date: 2017-02-25 12:24:11
tags:
	- golang
---

# 讨论群中关于切片的一个问题
## Q1：对slice的append无效
在群里有人提问，下述[代码][4]对slice的append无效

<!-- more -->

```golang
package main

import (
	"fmt"
)

func main() {
	list := make([]int, 2)
	add(list)
	fmt.Println(list)
}

func add(list []int) {
	list = append(list, 100)
}
```

执行结果：

	[0 0]


有人给出的解释是：
> 在add函数中执行append后，切片由于当前可用空间不足而执行了扩容，实际地址发送了变化，因此外层访问的list并非append后的list地址


## Q2：即使空间足够，slice不扩容，对slice的append还是无效
根据上面的解释，提问者修改了示例[代码][5]，以确保切片当前的可用空间足够，append时不进行扩容，仍然保持原有的list地址，但修改后的代码对slice的append仍然无效：
```golang
package main

import (
	"fmt"
)

func main() {
	list := make([]int, 0, 100) //leave enough space
	add(list)
	fmt.Println(list)
}

func add(list []int) {
  	list = append(list, 100)
}
```
执行结果：

	[]


# 细看slice

出于对上述现象的好奇，我对Go中slice的相关的内容进行了一些梳理

## 类型系统

在《The Go Programming Language》中，作者将Go中的数据类型做了如下分类：
> Go’s types fall into four categories: basic types, aggregate types, reference types, and interface types. Basic types, ... include numbers, strings, and booleans. Aggregate types—arrays ... and structs ... form more complicated data types by combining values of several simpler ones. Reference types are a diverse group that includes pointers ..., **slices** ..., maps ..., functions ..., and channels ..., but what they have in common is that they all refer to program variables or state indirectly, so that the effect of an operation applied to one reference is observed by all copies of that reference. Finally, ... interface types ... (from p.51)

但是，从上面的示例我们显然发现，slice并非是我们想象中的那样的“引用”类型。仔细阅读《The Go Programming Language》中关于slice介绍的章节(section 4.2)，我么会进一步看到：
> Updating the slice variable is required not just when calling append, but for **any function that may change the length or capacity of a slice or make it refer to a different underlying array**. ... In this respect, slices are not **"pure"** reference types but resemble an aggreate type... (from p.91)

从这里我们就可以看到，slice的确并非我们所预想的那样。

### 值语义和引用语义

Go中大多数类型都是基于值语义的：
* 基本类型：byte、int、bool、float32、float64和string等
* 复合类型：**数组(array)**、结构体(struct)、指针(pointer)等

另外还有4个类型比较特别，分别是：
* 数组切片(slice)
* 映射(map)
* 通道(channel)
* 接口(interface)

多数时候，上述4类型被认为具有引用语义的行为，而实际上如果细看，我认为这些类型还是值语义，只是这些类型的结构都是“具有指针成员变量”的结构体。因此在多数时候，将该类型变量传递后（实际上是结构体传递），能够通过指针成员变量在传递前后访问到相同的数据内容，因此，而被视为具有引用语义的类型。

总的来看，我认为Go语义类型是值语义！

## 内部结构

要弄明白为什么切片也是值语义，我们必须了解切片和数组的整体结构

### array的组织结构

和C/C++类似，Go中的数组也是一块连续的内存空间

![](http://og43lpuu1.bkt.clouddn.com/golang_slice_depth/png01_array.png)

如上图所示，一个[4]int的数组，其内存空间则是4个连续的int。Go中的数组是值语义。意味着每一次复制，每一次参数传递，都是将整个array的内存拷贝一遍。

### slice的组织结构

切片是“可变长”数组，为了做到动态的可扩容，实际上，切片的结构的内存空间如下图所示：

![](http://og43lpuu1.bkt.clouddn.com/golang_slice_depth/png02_slice-struct.png)

实际上他是一个具有三个域的结构，包括一个指向底层存放数据内容的数组指针ptr, 一个记录当前slice长度的len和一个记录当前slice容量大小的cap。

当对slice进行定义初始化时，则会为ptr赋予实际的内存空间地址，指向实际的底层数组：

![](http://og43lpuu1.bkt.clouddn.com/golang_slice_depth/png03_slice-1.png)

经过上面对slice结构的描述，我们就可以看到，对slice的赋值以及函数参数传递，实际上是对slice结构:

```golang
type slice struct {
	ptr *Elem
    len int
    cap int
}
```
的复制。由于slice赋值前后的两个变量，在不发生slice底层数组容量扩容（实际指针地址改变）的情况下，前后两个slice变量的ptr将指向同一块数组空间，因此，此时通过一个slice变量修改slice内某一位置的值，通过另外一个slice变量是可以访问到改变的。而如果修改了len或者cap变量，由于赋值前后两个slice变量是保存了len/cap各自的副本，因此在一个slice变量中的修改，将无法反应到另外一个slice的内容上

进一步的，我们也可以认为map本质上也是一个字典指针：

```golang
//具体实现
type MAP_K_V struct {
    // ...
}

//对外暴露的map类型
type map[K]V struct {
	impl *Map_K_V
}
```

同样的channel的实现实际上也是类似的结构，而接口的实现，则是他内部具有两个指针：

```golang
type interface struct {
    date *void
    itab *Itab
}
```

上述几个结构，都是在其暴露的结构体中具有指针类型，而使得看似该类型具有引用语音。而当我们明白了其内部构造，我们就会形成统一的认识，上述类型实际上是基于指针实现的值语义，进而在行为上具有类似引用语义的行为

## 对问题的分析

结合上面对slice的理解，我们可以发现，在上述Q2的代码描述中，实际上已经将100写入到list的空间了， 只是因为slice是值传递，在add里面，list的len修改为1， 但是出了add，list的len还是0，此时用Println是打印不出作者写入的100的（Println会根据len打印具体长度的数据）

为了验证我们的猜测，我们可以不直接初始化slice底层的array，而是通过基于一个现成的array来完成slice的构建，这个构建过程叫做切片化(slicing)
```golang
package main

import "fmt"

func main() {
	underlyArray := [5]int{}
	list := underlyArray[0:0]  // slicing

	fmt.Println(underlyArray[0])
	
	add(list)
	
	fmt.Println(list)
	fmt.Println(underlyArray[0])
}

func add(list []int) {
  	list = append(list, 100)
}
```
执行[代码][3]


	0
    []
    100

输出结果验证了我们的猜想，虽然赋值前后的len/cap是各自的副本，但由于slice的指针指向了同一块array，该array[0]在add函数中被修改为100，因此，在函数add外面，我们如果通过底层的array，直接访问array[0]，我们会看到，这个100已经被写入，而且可以被访问拿到。而由于外层函数的list保留了自己len=0, cap=0的副本，因此在打印时，由于底层数组长度被认为是0，而被放弃输出第0位的100

# 进一步思考

当在闭包closure之外修改了slice的内容，闭包内访问slice，能看到更新的内容吗？
答案和上述示例类似，No!

# 参考资料
[Go Slices: usage and internals][1]
[Arrays, slices (and strings): The mechanics of 'append'][2]

[1]: https://blog.golang.org/go-slices-usage-and-internals
[2]: https://blog.golang.org/slices
[3]: https://play.golang.org/p/PNvCKIeLek
[4]: https://play.golang.org/p/jWy3kP2A25
[5]: https://play.golang.org/p/G0WE7S22Wq