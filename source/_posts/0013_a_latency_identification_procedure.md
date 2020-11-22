---
title: 一次Golang程序延迟过大问题的定位过程
date: 2017-09-13 22:27:14
tags: 
	- golang
---



高吞吐、低延迟，一直是后台程序追求的目标。上周，我对PubSub系统进行了内存泄漏的分析定位，这周进一步对系统处理的延迟进行了分析，找到了引起处理延迟的关键节点。

<!-- more -->

老规矩，先来看看系统现象

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/01_before_upgrade_sub_p99.png)

上图是在我们对PubSub进行升级之前，PubSub系统在处理一次短连接的sub请求的耗时统计（客户端通过短连接发起一次http sub请求，获得数据，并最终关闭连接，真实的业务场景下，我们会使用http长连接，从而进一步降低每一次http连接建立、断开对每一次sub请求的overheader），从图上我们可以看出，短连接sub的耗时，大致在61ms左右（Y轴右侧坐标）。

在对PubSub系统进行升级改造后（我们使用了新的consumer group lib），再进一步对sub耗时进行统计，悲催的事情发生了...

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/02_after_upgrade_before_tunning_sub_p99.png)

我擦擦擦擦擦，sub延迟耗时尽然稳定在了312ms左右，整整多了有250ms！ @@，虾米情况啊...

虽然，暂时不知道究竟是什么引起了延迟的激增，但我知道，这是一次体验Golang调优的好机会，乐观点，事情总会解决的。

## CPUProfile耍起来

上周对内存泄漏进行定位的时候，我们已经看到了`net/http/pprof`和`runtime/pprof`的强大功能，他能帮我们采样统计程序运行时的很多数据，其中就包括了：cpuprofile。通过cpuprofile，我们就能看到程序运行耗时在哪里，将那些耗时最高的代码给我们列出来，这样一来，我们自然就能发现，是什么引起了延迟的过高。

通过`go tool pprof --seconds 30 http://127.0.0.1:9090/debug/pprof/profile`命令，我们可以抽取30秒的cpu采样统计信息。为发现延迟问题，我首先通过moni启动一个sub端，每2s中执行一次sub操作，以下是我们获取的cpuprofile信息：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/03_cpu_profile.png)

通过`top 20 -cum`按照`cum`字段进行排序（cum的含义是该函数以及该函数的内部调用的累计耗时，flat则是纯粹该函数，不包括该函数的内部调用函数），但是，我们看不错什么**明显突出**问题。

### 尝试使用火焰图(FlameGraph)

通过list方式，似乎不能很直观的观察、发现问题，那么咱们换个工具，使用时下比较流行的火焰图，我们在进一步观察一把：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/04_cpu_profile_flamegraph.png)

火焰图，是由uber的系统工程师Brendan Gregg弄出来的，他有一本很不错的书：[《Systems Performance：Enterprise and the Cloud》](https://www.amazon.com/Systems-Performance-Enterprise-Brendan-Gregg/dp/0133390098/ref=la_B004GG0SEW_1_1?s=books&ie=UTF8&qid=1505114273&sr=1-1)，在此基础上，uber开源了针对golang的火焰图生成工具：[go-torch](https://github.com/uber/go-torch)，火焰图实际上是在cpuprofile数据上的二次加工，他将大量的采样数据进行了归并统计，并以颜色深浅的形式体现cpu采样耗时的高低，使得使用者能够非常直观的，一眼看出，耗时的重点在哪里。

如上图所示，我将cpuprofile用火焰图的形式转换出来：torch.svg，但似乎还是没能看出有啥“蹊跷”之处。

#### 加大点压力试试

会不会是sub的频率太低，使得cpuprofile采样时，没能采样到? [SIGPROF信号每10ms触发一次](http://mycodesmells.com/post/profiling-with-pprof)，通过注册该信号的handler，cpuprofile执行10ms为间隔的统计数据采样:`runtime/proc.go`

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/23_sigprof.png)

from [pprof api ref](https://golang.org/pkg/runtime/pprof/):
> On Unix-like systems, StartCPUProfile does not work by default for Go code built with -buildmode=c-archive or -buildmode=c-shared. StartCPUProfile relies on the **SIGPROF** signal, but that signal will be delivered to the main program's SIGPROF signal handler (if any) not to the one used by Go. To make it work, call os/signal.Notify for syscall.SIGPROF, but note that doing so may break any profiling being done by the main program.

为此， 我加到了sub的频率，由原来的1个sub，提升到13个sub（13个不同的consumer group)同时进行sub，通过go-torch生成火焰图：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/05_cpu_profile_13sub_flamegraph.png)

采样到的信息更多了，但搜索一番，似乎还是没能找到可疑之处！

## 换个思路

PubSub系统要求一个consumer group中的client数量不能超过一个topic的partition数量，这就限制了我们不能像测试简单http请求一样不断的加大sub请求的数量，进而提高sub的压力，提升cpuprofile采样命中的概率。既然不能通过加压的方式换取更有用的cpuprofile，我们能否换一个思路呢？ 

之前调研微服务治理相关的分布式追踪系统时，分布式追踪的分段延迟统计功能给我留下了深刻的印象：

Google的StackDriver效果图：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/06_google_StackDriver_trace_details.png)


Amazon的X-Ray效果图：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/09_aws_xray_tracing.png)

从上面两张效果图，我们可以看出，分布式追踪系统能够很方便的显示每一段调用的延迟情况，帮助我们判定整个调用的延迟究竟是损耗在哪里。如果我们在分析golang程序处理延迟时，能有一个类似的工具，从调用链的角度帮我们串出整个调用处理逻辑上的延迟情况，那该有多好啊！

## Tracer很不错

我google，google，google...，终于找到了他：`go tool trace`。先来看张直观效果图：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/10_go_tool_trace.png)

重点关注Syscalls部分，里面的彩色进度条，就是在各时间点上运行于各Proc上的goroutine执行情况。

`go tool trace`究竟是何方神圣？golang官方对他的描述，简直是粗陋无比！https://golang.org/cmd/trace/  相信就算是你看完了，你也不会知道trace到底是什么。没关系，下面这段来自Dimtry Vyoukov的[介绍](https://docs.google.com/document/d/1FP5apqzBgr7ahCCgFO-yoVhk4YZrNIDNf9RybngBc14/pub)会让你明晰很多：

> Go has a number of profiling tools -- CPU, memory, blocking profilers; GC and scheduler tracers and heap dumper. However, profilers provide only aggregate information, for example, how much memory in total was allocated at this source line. Tracers provide very shallow information. Heap dumper gives detailed per-object information about heap contents. <font color="#FF0000">But there is nothing that gives detailed non-aggregate information about program execution -- what goroutines do execute when? for how long? when do they block? where? who does unblock them? how does GC affect execution of individual goroutines?
> 
The goal of the tracer is to be the tool that can answer these questions.</font>

简言之就是，cpuprofile做的是采样、聚合的工具，他告诉你的是在这段时间内，总共这个函数被调用了多少次，耗时多少，但是这些工具没有从时间的角度来串联，而tracer则是从时间的角度来串联goroutine的行为，告诉你，在什么时候，goroutine做了什么，以及执行了多长时间，阻塞在哪里。


### 分析过程

有了得力的工具，下面就是深入的分析了

#### Step1: 获取trace数据

要获取trace数据，我们还是通过`net/http/pprof`提供的接口，只是之前是获取`/debug/pprof/heap`、`/debug/pprof/goroutine`等，这次是获取`/debug/pprof/trace`信息，通过：`curl -XGET  "http://127.0.0.1:10194/debug/pprof/trace?seconds=30" -o 002_trace_2017_09_08.out`我们将获取一个30秒的trace数据(trace_02.out)，通过`go tool trace 002_trace_2017_09_08.out`，我们将开启tracer的探索之旅：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/11_30s_trace_overview.png)

从上图我们可以看到，在图标的正中间，是一段有统计数据的30s区间，这里就有我们需要的数据


#### Step2: 定位一个事件周期

既然是要分析一次sub的延迟究竟是消耗在了哪里，那么首先我们就需要从tracer上找到一个sub究竟是从哪里到哪里。sub是从一次http请求开始，到一次http响应结束。从tracer上我们首先去定位Network有数据的地方：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/12_begin_with_network_data_flow.png)

从这里开始，放大这个时间点的goroutine信息，我们将找到我们需要的sub开始的http handler

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/12_http_handler.png)

从goroutine当时的调用栈，我们可以明确的确认，这就是一个sub请求开始的地方

进一步的，我们需要确认一个sub请求的结束(http response结束)的地方。既然我们已经查到http请求开始的地方是在2992号Goroutine上，那我们可以通过tracer去过滤查找2992号Goroutine的踪迹，最终我们可以找到Goroutine 2992结束的地方，基本上就能对应到http response结束的地方

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/13_http_handler_end.png)

我们可以看看整个Goroutine 2992的生命周期

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/14_http_handler_range.png)

307ms和客户端统计的312ms的延时，非常接近。


#### Step3: 深入细节

已经定位了一次sub的整个周期，那么我们可以进一步的深入进去，观察在这个周期中究竟发生了什么

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/15_range_target.png)

放大整个sub周期，我们会看到，真正Goroutine活动被分隔为2圆圈画出的两部分，在这两部分之外，几乎没有goroutine活动。进一步的，我们分别观察这两个区域的goroutine活动细节


##### consumer启动时，release等待100ms

在第一个圆圈区域的最末尾，我们会看到如下图所示的goroutine活动：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/16_target_1.png)

Goroutine 3141的活动，是整个区域最后的活动，通过查看此时Goroutine 3141的堆栈信息，我们可以看到：`sarama-cluster.(*Consumer)/release:427`，这一行是在干什么？

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/16_target_1_code_427.png)

这是一个无法避开的waiting！waiting了多久呢？我们得查看`DwellTime`:

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/17_targe_1_time_config.png)

在我们的代码中，被设置成了100ms，也就是说，这里被我们强行设置成了等待100ms。进一步的，我们通过tracer上的Goroutine活动间隔，进一步确认，的确是被waiting了100ms

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/18_target_1_over.png)


##### consumer启动时，fetchOffset前等待200ms

类似的，我们立马发现在第二个活动区域，出现了类似的waiting，只是这次wait的是200ms(DwellTime * 2):

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/19_target_2.png)

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/20_targe_2_code.png)

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/21_target_2_range.png)

基本上，我们已经可以确定，就是设置的`DwellTime`值过大，导致cosnumer在创建时，延迟过高，进而影响了整个短连接sub的延迟。


### 解决方案

很直观，为减低延迟，我们需要做的就是降低DwellTime。为什么代码中需要有两个DwellTime的等待呢？从代码上的注释我们可以看出，实际上，这是sarama-cluster做的一个保险措施：consumer在发生rebalance的时候，会通过这个DwellTime等待其他工作goroutine标记最后一次处理的offset，并在等待DwellTime之后，做最后一次offset提交动作。对于PubSub而言，PubSub支持两种offset提交：1. AutoCommit，每次获取消息，在PubSub端就直接完成OffsetCommit；另外一种是DelayCommit，由下一次Sub请求，来标记OffsetCommit。对于第一种offset提交，整个处理过程是非常快的（因为只涉及系统内数据的传递之后，便完成commit），此时，完全没有必要等待100ms，而对于第二种offset提交，等待多长，都不合适，因为我们无法确定两次sub的时间间隔（由于业务系统在sub之后还有自己的处理逻辑，而这个处理逻辑的时间跨度完全不定）因此，我们也无法确定一个等待时间。基于上述分析，我们可以看出，DwellTime设置过长，对PubSub而言，意义不大。DwellTime设置短带来的可能副作用是一条消息被重复消费。而PubSub本来就是支持at least once语义，即要求业务系统在业务逻辑上做好幂等处理，因此基于上述考虑，我们调整wellTime到一个较小值：5ms，只要涵盖在AutoCommit下rebalance切换的顺利。基于5ms的设置，重新测试，观测sub延迟结果


![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/22_after_tunning.png)

2个篮圈，分别表示调整前的312ms和调整后的27ms，延迟有效降低。


### 为什么cpuprofile不好使？

回过头，我们再来看看为什么cpuprofile不好使呢？ 

#### golang runtime scheduler

从golang 1.1开始，Dmitry Vyukov重新设计了[golang runtime的scheduler](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit#!)，进而引入了G-P-M模型。

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/27_scheduler_G_P_M.jpg)

其中，G代表一个goroutine，他包含了goroutine的执行栈信息、goroutine的状态以及其他和goroutine调度相关的重要信息。P代表逻辑的processor，是一个调度的上下文(context)。而M代表的是系统线程，他是真正执行计算的资源。M在绑定有效的P后，进入schedule循环，从P的队列中获取待调度G，从而执行G的逻辑

常规情况下，M, P, G之间的关系大致如下图所示：

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/24_normal_scheduler.jpg)

灰色部分的G目前正在等待被调度，他们被存放在一个个runqueue中。每次在程序中执行go语句，就会创建一个G，并将该G加入到runqueue队列的尾部。当P发生goroutine调度时，他会从runqueue的头上取出一个G，并将G的状态信息装载到自己的上下文中，并开始执行该G的逻辑。

当有G在执行过程中，陷入系统调用时，M, P, G的关系会有些变化

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/25_syscall_scheduler.jpg)

当G0需要执行系统调用时，scheduler会将G0从上下文P中剥离出来，逻辑上的G0和对应的M0一同进入系统调用的等待，而P将重新attach到新的M1，进而执行其他待执行的goroutine，直至G0的系统调用返回，M0则有可能steal一个新的P来执行G0，又或者将G0加入到P的runqueue中


#### cpuprofile只采集P上运行的信息

从上面对schedule的描述，我们可以看出，当G0陷入系统调用时，G0与P已经剥离开来。而当我们执行cpuprofile的时候，实际上是在统计当前在各个P上执行的G，分别处于什么状态，正因为此时G0并不在任何P上，因此，我们通过cpuprofile实际上是无法统计到关于G0进入系统调用的信息

![](https://lday-me-1257906058.cos.ap-shanghai.myqcloud.com/0013_a_latency_identification_procedure/img/26_cpuprofile_useless_for_syscall.png)

采样只会统计到处于P上的fun1以及fun2的信息，而无法统计到syscall部分的信息。这也就是为什么，我们在之前使用cpuprofile时，未能查看到关于进入waiting的信息




## 结论

一次比较简单的latency调优过程，从中我们可以看到，cpuprofile、trace均能帮助我们定位latency相关的问题，但是两者各有侧重：

* cpuprofile: 侧重于统计程序各goroutine自身的运行状况，更加适用于分析针对cpu密集型逻辑导致的latency过高问题
* trace：以goroutine整个生命周期的事件处理为主线，充分展现了各goroutine生命周期内的状态变化，能够很好的反映系统调用下goroutine的切换逻辑，更加适用于IO密集型(系统调用密集型)逻辑导致的latency过高问题

## 参考资料

* [Go Profiling and Optimization](https://www.youtube.com/watch?v=N3PWzBeLX2M)
* [go tool trace](https://making.pusher.com/go-tool-trace/)
* [Fighting latency: the CPU profiler is not your ally](https://www.youtube.com/watch?v=nsM_m4hZ-bA)
* [Go's execution tracer](https://www.youtube.com/watch?v=mmqDlbWk_XA)
* [The Go scheduler](https://morsmachine.dk/go-scheduler)
* [Go scheduler: Ms, Ps & Gs](https://povilasv.me/go-scheduler/)
* [也谈goroutine调度器](http://tonybai.com/2017/06/23/an-intro-about-goroutine-scheduler/)




