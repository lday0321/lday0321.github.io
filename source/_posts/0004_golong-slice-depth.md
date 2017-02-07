---
title: 细看Go中的切片(slice)
date: 2017-02-06 14:59:11
tags:
	- golang
---

# 讨论群中关于切片的一个问题
## Q1：对slice的append无效
在群里有人提问，下述[代码][4]对slice的append无效
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

## 数据类型

在《The Go Programming Language》中，作者将Go中的数据类型做了如下分类：
> Go’s types fall into four categories: basic types, aggregate types, reference types, and interface types. Basic types, ... include numbers, strings, and booleans. Aggregate types—arrays ... and structs ... form more complicated data types by combining values of several simpler ones. Reference types are a diverse group that includes pointers ..., slices ..., maps ..., functions ..., and channels ..., but what they have in common is that they all refer to program variables or state indirectly, so that the effect of an operation applied to one reference is observed by all copies of that reference. Finally, ... interface types ...

## 内部结构

### array的组织结构

### slice的组织结构


## 对问题的分析
```golang
package main

import "fmt"

func main() {
	underlyArray := [5]int{}
	list := underlyArray[0:0]

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

我的回复：
> 你示例add函数里面，实际上已经将100写入到list的空间了， 只是因为slice是值传递，在add里面，list的len修改为1， 但是出了add，list的len还是0，此时用Println是打印不出你写入的100的。我改了下示例，你可以看到underlyArray[0]已经被修改成100了：

# 进一步思考

当在闭包closure之外修改了slice的内容，闭包内访问slice，能看到更新的内容吗？

# 参考资料
[Go Slices: usage and internals][1]
[Arrays, slices (and strings): The mechanics of 'append'][2]

[1]: https://blog.golang.org/go-slices-usage-and-internals
[2]: https://blog.golang.org/slices
[3]: https://play.golang.org/p/PNvCKIeLek
[4]: https://play.golang.org/p/jWy3kP2A25
[5]: https://play.golang.org/p/G0WE7S22Wq