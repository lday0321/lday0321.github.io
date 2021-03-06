---
title: 一次Golang程序内存泄漏分析之旅
date: 2017-09-02 02:49:14
tags: 
	- golang
---


最近在开发升级PubSub系统，目标是支持更新版本的kafka（从原来的支持kafka 0.8.2.2升级到支持较新版本的kafka 0.10.1.1）。由于kafka在0.9版本上对consumer group相关的结构进行了[重构](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Client-side+Assignment+Proposal)，原来使用的基于zk进行rebalance的[consumer group client:wvanbergen/kafka](https://github.com/wvanbergen/kafka)已经不再适用。为此，我们调研并最终选用了支持新版kafka consumer rebalance的[consumer group client:sarama-cluster](https://github.com/bsm/sarama-cluster)。功能开发已基本结束，目前已进入系统压力测试阶段。

<!-- more -->

花了二周时间，用golang完成了一个分布式的测试工具框架，满足我对PubSub系统大规模分布式并发压力测是的需求。同时，这段时间，我已经将PubSub系统部署起来，并通过一个监控程序每隔30s向目标topic发送pub和sub请求。一周后，当我打开监控面板，“期待已久”的幺蛾子出现了：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/01_2w_memory_leak.png)

从上图我们可以看出，在两周多的时间里，系统的堆内存画出了一条"完美"的增长曲线，WTF！

怀着无比激动的心情， 我开始了这次内存泄漏的分析之旅

## 准备工作

工欲善其事，必先利其器。要想方便，快捷的定位、解决问题，首先你得有一套好使，方便的工具。

### 测试工具

我花2周时间写的分布式自动化测试工具正好派上了用场。简单的介绍一下我的分布自动化测试工具：“摸你”(moni)。由于时间原因，目前moni只实现了一些基础功能，不过，对于这次的目标已经够用了。moni是一套主-从结构的分布式测试工具。大致结构如下所示：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/25_moni_arch.png?imageView2/2/w/700)

moni分为三部分：
* mrunner，用于执行具体的测试任务，保存执行状态及结果。
* mman，中心管理节点。用于向所有mrunner发布任务，并根据需求向mrunner抓取并汇总执行结果。同时提供RESTFul api，为终端用于提供接口，用于发布各类命令。
* mctl，测试执行人操作工具，通过该工具调用mman的RESTFul接口，并最终将各测试命令下发至各mrunner。
* magent，暂未实现。用于支持在大规模分布式环境下多台mrunner的自动升级和自动起、停，目前通过脚本半自动化来完成

目前，mctl支持三类命令：
1. status，收集查看moni各节点的状态
2. pub，执行pub测试，
3. sub，支持sub测试


各命令又包含若干自参数，用于定制各类测试场景，通用的一些参数包括：
* -d: duration，用于设定测试时间长度
* -i: interval, 用于设定请求间隔
* -c：concurrency，用于设定并发量
* -cm: connection mode，用于设定链接的类型（长链接or短连接）
* 等等


mman与mrunner之间保持TCP长连接，mrunner会在与mman的连接断开后主动每间隔一定时间便尝试重连，并在每次连接成功后执行runner注册动作。mman同时作为一个http server，向mctl暴露HTTP的RESTFul接口，接收来自mctl的指令，并返回结果

查看mrunner整体情况：
![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/02_mctl_status.png)

下图则是我在向13个runner下发了sub测试指令（持续12小时，每2s执行一次sub，并发量为1）之后，查看sub执行情况：
![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/03_mctl_sub_status.png)

有了moni这套工具， 我就能很方便的模拟测试各类场景，帮助我重现系统的问题。

### 数据采集、汇总工具

仅仅有了模拟工具显然是不够的，作为系统自身，你还需要有一套完整的数据采集、汇总工具，通过这些工具，我们能收集系统的各类指标，并将这些指标最终展现出来，方便我们观察系统状态，定位系统问题。PubSub系统基于Golang实现，golang实际上提供了非常好的工具来帮助我们采集各类指标

#### 系统监控统计级别数据指标

1. goroutine数量
2. 堆(heap)内存使用量
3. 栈(stack)内存使用量
4. 等等

这些数据我们都将进行统计，并最终写入到influxdb中，并经由grafana展示出来

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/04_golang_grafana.png)

这些数据可以通过[runtime包](https://golang.org/pkg/runtime/)内的接口获得，当然，为获得各类型的统计值，使用封装好的库会是更加方便的选择：[rcrowley/go-metrics](https://github.com/rcrowley/go-metrics)。

#### 问题定位所需的详细数据

上述监控级指标，只能告诉你系统是不是出问题了，但是你很难从上述指标中定位出具体是哪里出了问题。在分析内存泄漏时，你不但要确定是否有内存泄漏，你还需要在发现存在内存泄漏时明确定位出在哪里出现了内存泄漏！ 这时候就需要使用一些更细的指标信息。golang已经为我们做好了准备，你可以：

* 获取系统实时堆内存分配详细信息
  具体到这块内存是在哪里分配的

```
heap profile: 9: 8474736 [13: 8556768] @ heap/1048576
2: 3145728 [2: 3145728] @ 0x791484 0x798931 0x45b621
#	0x791483	github.com/ffan/xxx/vendor/github.com/samuel/go-zookeeper/zk.(*Conn).recvLoop+0x53	/home/chenlei106/Works/GoWorks/PubSub_2/src/github.com/ffan/xxx/vendor/github.com/samuel/go-zookeeper/zk/conn.go:679
#	0x798930	github.com/ffan/xxx/vendor/github.com/samuel/go-zookeeper/zk.(*Conn).loop.func2+0x40	/home/chenlei106/Works/GoWorks/PubSub_2/src/github.com/ffan/xxx/vendor/github.com/samuel/go-zookeeper/zk/conn.go:424

...

# runtime.MemStats
# Alloc = 13127416
# TotalAlloc = 15482888
# Sys = 21641464
# Lookups = 126
# Mallocs = 51022
# Frees = 33571
# HeapAlloc = 13127416
# HeapSys = 15564800
# HeapIdle = 1277952
# HeapInuse = 14286848
# HeapReleased = 0
# HeapObjects = 17451
# Stack = 1212416 / 1212416
# MSpan = 54416 / 81920
# MCache = 38400 / 49152
# BuckHashSys = 1445508
# GCSys = 796672
# OtherSys = 2490996
# NextGC = 101353464
# PauseNs = [838359 1139459 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0]
# NumGC = 2
# DebugGC = false
```

* 获取系统实时所有goroutine调用栈信息
  具体到这个goroutine是在哪里启动的，以及当前是在干什么

```
goroutine 1 [chan receive, 2 minutes]:
github.com/ffan/xxx/cmd/kateway/gateway.(*Gateway).ServeForever(0xc4200aa0a0)
	/home/chenlei106/Works/GoWorks/PubSub_2/src/github.com/ffan/xxx/cmd/kateway/gateway/gateway.go:339 +0x65
main.main()
	/home/chenlei106/Works/GoWorks/PubSub_2/src/github.com/ffan/xxx/cmd/kateway/main.go:101 +0x287

goroutine 17 [syscall, 2 minutes, locked to thread]:
runtime.goexit()
	/home/chenlei106/Tools/Dev_Tools/go_1_8/src/runtime/asm_amd64.s:2197 +0x1
```
  
  
  
* 获取系统实时堆内存调优辅助统计信息
  具体是在哪里分配了多少内存，以及top N分别是哪些，甚至是每个内存分配的来源图
![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/05_go_tool_pprof.png)


![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/06_go_tool_pprof_profile.png)

让我们一个一个来看。

##### 获取系统实时堆内存分配详细信息

 为了获取系统实时堆内存分配的详细信息，需要使用[net/http/pprof包](https://golang.org/pkg/net/http/pprof/)，通过该包，只要在我们的http server上注册指定的路由路径，就可以在访问路径时获取到实时的heap内存分配详细信息。

具体做法是：
```golang
//引入pprof
import "net/http/pprof"

//在http router上加入
this.debugMux.HandleFunc("/debug/pprof/", http.HandlerFunc(pprof.Index))
```

在启动server后，直接可以通过：
`curl -XGET  "http://192.168.149.150:10194/debug/pprof/heap?debug=2"` 获取heap内存的详细信息，其中10194是你开启的http server的端口，debug=2意味着需要输出详细信息

##### 获取系统实时所有goroutine调用栈信息

在完成`this.debugMux.HandleFunc("/debug/pprof/", http.HandlerFunc(pprof.Index))`注册后，实际上我们不光可以获取heap信息，我们也同样可以获得goroutine信息，只是最终的http路径不同而已，通过`curl -XGET  "http://192.168.149.150:10194/debug/pprof/goroutine?debug=2"`拿到的就是goroutine的详细信息

##### 获取系统实时堆内存调优辅助统计信息

基于上述由目标程序暴露的http接口，使用golang提供的工具，我们就能对统计信息进行分析：`go tool pprof -inuse_space http://192.168.149.150:10194/debug/pprof/heap`，进入pprof交互模式后，可以通过`top`, `tree`等进一步查看统计信息，同时，也可以通过`png`命令，将内存信息输出成图片，以图片的形式显示内存的分配、占用情况


##### 其他说明

对于go pprof工具的具体使用，可参考golang官方blog的文章：[Profiling Go Programs](https://blog.golang.org/profiling-go-programs)，或者郝林的中文说明：[go tool pprof](https://github.com/hyper0x/go_command_tutorial/blob/master/0.12.md)

## 模拟场景，重现问题

### 场景一：完全仿照之前的测试，通过长连接执行Pub/Sub调用

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/07_long_conn_pub_sub.png)

经过两轮测试，我们发现，内存占用“似乎”有一定的变化，但又不是那么明显，此时我们无法确定是否真重现了内存泄漏的问题。


### 场景二：改用短连接执行Pub/Sub调用

仔细分析前两周PubSub系统的静默运行行为，我们发现，实际上当时的Pub采用的是长连接，而Sub实际上采用的是短连接模式。为此，我们调整了moni的测试连接模式，改成短连接模式

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/08_short_conn_2r_test.png)

如上图所示，在调整为短连接模式后，我们又执行了两轮测试，其中第一轮测试初始化时5个Pub，5个Sub，每个客户端分别以2s为间隔执行sub，在执行了40分钟后，我又进一步加大了压力。同理，第二轮测试与第一轮测试类似。从图上我们可以看出，换成短连接后，内存泄漏的现象已经被清晰的展示出来，其中每轮测试，内存泄漏量大致在20M之上


### 场景三：单独使用Pub调用，确认是否由Pub调用引起内存泄漏

为进一步确认是由Pub调用引起的内存泄漏，还是由Sub调用引起的内存泄漏，我们单独对Pub进行了测试，如下图所示，在单独执行pub的测试中，我们发现，pub对内存泄漏没有影响

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/09_short_conn_pub.png)

### 场景四：单独使用Sub调用，确认是否由Sub调用引起内存泄漏

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/10_short_conn_sub.png)

在单独使用sub进行测试的过程中，我们明显发现了内存泄漏


### 场景五：单独使用Sub调用，长时间压力测试

为进一步验证在sub上存在内存泄漏，我们对PubSub系统进行了2组长时间测试，每组测试10小时，12个sub，每2s执行一次sub

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/11_short_conn_sub_longterm.png)

### 总结

通过上述几轮的测试，基于收集到的数据，我们基本上已经明确：内存泄漏来自于短连接Sub操作。后续我们则需要针对性的对短连接sub的执行逻辑以及具体数据变化进行分析了

## 获取数据，比对分析

golang是一门自带gc的语言，这也就意味着大多数情况下我们不需要过多的关心何时申请内存，何时释放内存。但是，即使是有gc，也不能排除像我们这次遇到的内存泄漏的问题。在有gc的场景下，内存泄漏主要是有一些对象，被“意外”的勾住，引用计数不为0，导致gc错误的认为对象正在使用，而无法回收。要让内存对象在异常情况下引用计数不为0，我认为无外乎有两种情况：
1. 有goroutine泄漏，goroutine“飞”了，zombie goroutine没有结束，这个时候在这个goroutine上分配的内存对象将一直被这个僵尸goroutine引用着，进而导致gc无法回收这类对象，内存泄漏。
2. 有一些全局（或者生命周期和程序本身运行周期一样长的）的数据结构意外的挂住了本该释放的对象，虽然goroutine已经退出了，但是这些对象并没有从这类数据结构中删除，导致对象一直被引用，无法被回收。

### 确认goroutine是否泄漏

为找出内存泄漏的来源，我们首先来确认goroutine是否存在泄漏问题。下图是我们整个测试周期内，PubSub系统goroutine的变化情况：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/12_goroutine_cnt.png)

从上图我们可以看出，整个测试过程中，goroutine数量会随着测试的推进而变化，但不会持续增长，同时，在sub测试结束后，goroutine的数量又会回落到初始化持平状态，goroutine数量基本没有变，通过goroutine数据的观察，我们可以确定，不存在goroutine泄漏。同时我们可以看到在之前有pub参与时，goroutine提升到了1k以上，但是在只有sub参与测试中，goroutine数量维持在了一个较低的水平（85左右）。之所有pub参与时，goroutine数量会上升，是因为PubSub系统对Pub调用构建了对象池，虽然pub动作是短连接，但是在PubSub系统中，和后台kafka连接的Broker对象并未关闭，这些对象被放在了对象池中，并未回收。而这些对象的生命周期内有存活的内部goroutine，因此，在有pub的测试中goroutine的数量明显更多。在有pub参与的测试结束后，我们通过`curl -XGET  "http://192.168.149.150:10194/debug/pprof/goroutine?debug=2"`拿到goroutine调用栈的信息时，可以看到很多类似的信息：
```
goroutine 9360181 [chan receive]:
github.com/ffan/xxxx/vendor/github.com/Shopify/sarama.(*Broker).responseReceiver(0xc42659c000)
	/home/chenlei106/Works/GoWorks/PubSub_2/src/github.com/ffan/xxxx/vendor/github.com/Shopify/sarama/broker.go:484 +0xfa
github.com/ffan/xxxx/vendor/github.com/Shopify/sarama.(*Broker).(github.com/ffan/xxxx/vendor/github.com/Shopify/sarama.responseReceiver)-fm()
	/home/chenlei106/Works/GoWorks/PubSub_2/src/github.com/ffan/xxxx/vendor/github.com/Shopify/sarama/broker.go:148 +0x2a
github.com/ffan/xxxx/vendor/github.com/Shopify/sarama.withRecover(0xc426210ef0)
	/home/chenlei106/Works/GoWorks/PubSub_2/src/github.com/ffan/xxxx/vendor/github.com/Shopify/sarama/utils.go:46 +0x43
created by github.com/ffan/xxxx/vendor/github.com/Shopify/sarama.(*Broker).Open.func1
	/home/chenlei106/Works/GoWorks/PubSub_2/src/github.com/ffan/xxxx/vendor/github.com/Shopify/sarama/broker.go:148 +0x910
```


### 确认内存泄漏的具体位置

gorutine既然没有泄漏，那内存泄漏只可能是第二种可能：有一些全局（或者生命周期和程序本身运行周期一样长的）的数据结构意外的挂住了本该释放的对象。为找到这个数据结构，我们需要先进一步所以范围：需确认被挂住的内存到底是在哪里

通过`go tool pprof`工具，以及从`curl -XGET  "http://192.168.149.150:10194/debug/pprof/heap?debug=2"`，我们获得了我们想要的数据。首先，我们通过`go tool pprof`工具查看到如下信息：

```
Entering interactive mode (type "help" for commands)
(pprof) top 20
939.05MB of 949.56MB total (98.89%)
Dropped 464 nodes (cum <= 4.75MB)
      flat  flat%   sum%        cum   cum%
  486.54MB 51.24% 51.24%   912.56MB 96.10%  github.com/ffan/xxxx/vendor/github.com/rcrowley/go-metrics.newStandardMeter
  150.51MB 15.85% 67.09%   150.51MB 15.85%  github.com/ffan/xxxx/vendor/github.com/rcrowley/go-metrics.NewEWMA1
  140.01MB 14.74% 81.83%   140.01MB 14.74%  github.com/ffan/xxxx/vendor/github.com/rcrowley/go-metrics.NewEWMA15
  135.51MB 14.27% 96.10%   135.51MB 14.27%  github.com/ffan/xxxx/vendor/github.com/rcrowley/go-metrics.NewEWMA5
   26.48MB  2.79% 98.89%   939.05MB 98.89%  github.com/ffan/xxxx/vendor/github.com/rcrowley/go-metrics.NewMeter
         0     0% 98.89%   939.05MB 98.89%  github.com/ffan/xxxx/vendor/github.com/Shopify/sarama.(*Broker).Open.func1
         0     0% 98.89%   612.02MB 64.45%  github.com/ffan/xxxx/vendor/github.com/Shopify/sarama.getOrRegisterBrokerMeter
         0     0% 98.89%   939.05MB 98.89%  github.com/ffan/xxxx/vendor/github.com/Shopify/sarama.withRecover
         0     0% 98.89%   939.05MB 98.89%  github.com/ffan/xxxx/vendor/github.com/rcrowley/go-metrics.(*StandardRegistry).GetOrRegister
         0     0% 98.89%   939.05MB 98.89%  github.com/ffan/xxxx/vendor/github.com/rcrowley/go-metrics.GetOrRegisterMeter
         0     0% 98.89%   939.05MB 98.89%  reflect.Value.Call
         0     0% 98.89%   939.05MB 98.89%  reflect.Value.call
         0     0% 98.89%   939.05MB 98.89%  runtime.call32
         0     0% 98.89%   948.06MB 99.84%  runtime.goexit
```
从heap内存分布来看，大头出自：
```
939.05MB 98.89%  github.com/ffan/gafka/vendor/github.com/Shopify/sarama.(*Broker).Open.func1
```

通过在交互模式下生成png图片，我们进一步来看：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/13_08_30_after_test_profile009.png)

上图已经很明确的显示出，是在saram.Broker.Open函数中，New了各类Meter，而这些Meter就是内存泄漏的地方。同时，在我们通过`curl -XGET  "http://192.168.149.150:10194/debug/pprof/heap?debug=2"`收集的heap详细信息中，我们也可以看到：
```
#	0x69e3cc	github.com/ffan/xxxx/vendor/github.com/Shopify/sarama.(*Broker).Open.func1+0x101c	/home/chenlei106/Works/GoWorks/PubSub_2/src/github.com/ffan/xxxx/vendor/github.com/Shopify/sarama/broker.go:118
#	0x69e51c	github.com/ffan/xxxx/vendor/github.com/Shopify/sarama.(*Broker).Open.func1+0x116c	/home/chenlei106/Works/GoWorks/PubSub_2/src/github.com/ffan/xxxx/vendor/github.com/Shopify/sarama/broker.go:121
#	0x69dd6e	github.com/ffan/xxxx/vendor/github.com/Shopify/sarama.(*Broker).Open.func1+0x9be	/home/chenlei106/Works/GoWorks/PubSub_2/src/github.com/ffan/xxxx/vendor/github.com/Shopify/sarama/broker.go:148
```
非常多的内存对象，来自broker.go:118/121/148中Broker.Open调用的meter创建

### 深入分析内存泄漏的逻辑

从Broker的Open来看，Broker是通过起goroutine来申请的资源，最开始我是怀疑是由于Broker起了goroutine，但是没有close goroutine，导致内存泄漏。

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/14_sarama_broker_open.png)

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/15_sarama_broker_open_meter.png)

从上面对goroutine数据的观察我们可以看出，在sub时，并没有goroutine泄漏。实际上，为了进一步确认Broker的关闭情况，我还开启了sarama的log，在broker open和close的地方进行了打点统计，统计的结果也的确是所有open的broker，最终均close掉。

我们只能进一步分析代码。说实话，分析到这里，对着代码走了一圈，也没看出有啥问题来（关键是代码那么多，哪里都有可能出问题，隐隐觉得应该是被什么东西挂住了，就是找不到，到底是什么东西挂住了这些资源！），倒腾了有一天，反复debug，反复比对数据，诶嘿，终于找到了！

其实最终的定位有些运气成分。在尝试了各种分析无果之后，无奈之下，我只能再次进行测试，收集数据，期望能从测试中发现一些不一样的痕迹来。这次测试和往期不同，这次测试，我首先把环境清理的干干净净，PubSub系统整个停掉，重新启动，并且，在执行任何测试之前，我先通过`http://192.168.149.150:10194/debug/pprof`收集了一把heap和goroutine的数据。然后再执行sub压力测试。在测试完之后，再度收集heap，goroutine的数据。前后比较了一番，我发现了goroutine上的蛛丝马迹！

在比对测试前后goroutine的信息时，我发现在测试结束后，尽然多出了一个非常可疑的goroutine：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/16_goroutine_compare.png)

之所以非常可疑，是因为这个tick的gorutine，就是在处理Meter相关的逻辑，和我们从heap信息中看到的泄漏数据非常吻合。为进一步确认，顺着代码，仔细看看：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/17_meter_tick_goroutine.png)

隐约感觉这个`ma.meters`就是勾住内存对象的某个全局变量，顺着函数，这货到底从何而来！

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/18_meter_global_list.png)

`arbiter`是一个彻头彻尾的全局变量！

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/19_meter_global_list_2.png)

我擦擦擦，胜利就在眼前了！如果在`arbiter.meters = append(arbiter.meters, m)`之后，没有remove，那就有问题了！看着list append的动作，估计是没有remove，要是需要remove，一般会用map吧。搜了一圈代码，果真没有！ 那应该就是这里，导致memory leak了。


## 确认问题，解决方案

问题基本上确定了， 为进一步肯定自己的推测，这个时候，我到https://github.com/rcrowley/go-metrics 查查issue list，看看有没有人有类似的问题

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/20_leak_issue.png)

果然，赫然在列！进一步，我们会看到sarama中和这个相关的issue:
问题根源：go-metrics存在内存泄漏！
https://github.com/rcrowley/go-metrics/issues/201

引起在sarama频繁开启，关闭broker时，导致内存不断泄漏
https://github.com/Shopify/sarama/issues/897

go-metrics内存泄漏，证据确凿！！！

尽然已经发现问题了，下一步咱们得想办法解决掉他。从sarama相关issue的说明来看：
>It does not seem possible in go-metrics to unregister the meter and get that reference removed (even if we do not unregister the metrics in Sarama today)...

>Note that we do not use timer metrics but that would result in the same issue as the StandardTimer struct uses a meter under the hood...

>Looks like this needs to be fixed in go-metrics first and we would also have to use Registry#Unregister(string) or something

换句话说，sarama得等go-metrics提供了修复内存泄漏的Unregister接口后，在必要的地方(Broker Close的时候)Unregister，才能避免内存泄漏。那有没有别的workaround呢？仔细分析sarama中metrics相关逻辑，我们会看到，sarama是在使用metrics做一些统计信息。其实，这些统计信息对于我而言，完全可以不用，如果不用，把metrics禁掉，是否可行呢？从sarama的代码中我找到的思路：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/21_use_nil_metrics.png)

当我们将`metrics.UseNilMetrics`设置为`true`后，我们会看到，所有类型的metrics的创建，实际上都是返回一个空的metrics:

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/22_nil_meter.png)

这样一来，全局的`arbiter.meters`链表就不会勾住任何内存对象，进而内存泄漏就不存在了。


## 总结

这次一次有意思的内存泄漏分析之旅。以前使用C/C++开发，经常也要进行内存泄漏的定位，相对而言驾轻就熟些，第一次golang的内存泄漏定位，经历了最开始的兴奋，到各种尝试无果的失望，再到无心插柳的灵光乍现，直到最终解决问题的满足，在解决问题的同时，也对自己收获不少而高兴。回顾一下，整个分析过程，也许做到一下几点，会有助于今后遇到类似问题，更加快速的解决问题：

1. 测试工具要跟上，方便的测试工具有助于迅速重现问题，节省测试时间
2. 数据收集、展现手段要够详细，既要有直观的数据统计，通过图表能够直观的帮助你察觉是否有问题，又要能通过golang提供的pprof工具集接口进一步细化数据，深入分析
3. 定位要明确，在没有找到根源时，需要逐一排除。实际上在本次分析定位过程中，我一直怀疑有goroutine泄漏，花了较多时间来确认goroutine没有问题。
4. 需要有足够的耐心，不断的尝试，做完善的对比。这次最终能够发现问题，也多亏自己不断的尝试不同的测试方案和手段，做比较充分的数据比对
5. 找准目标仔细分析代码

补充说明一点，实际上在确认内存在Broker.Open泄漏后，我就一直在“仔细”阅读代码。但是为啥就一直没看到那个`arbiter.meters = append(arbiter.meters, m)`呢。回过头，再给自己的“不仔细”找个理由，请仔细观察一下两图：

本次读代码使用的VSCode：
![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/23_vscode_GetOrRegsterMeter.png)

另外使用的Goglang:
![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0012_a_memory_leak_detection_procedure/img/24_goglang_GetOrRegsterMeter.png)


看到差异了吗，vscode对NewMeter尽然没高亮！自己读代码的时候，以为NewMeter是变量，没注意他原来是个函数... -,-b，此时，一个论题在我脑海中浮现：《论优秀IDE对分析问题的重要性》

over。