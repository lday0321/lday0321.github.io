---
title: 什么是内存屏障(Memory Barriers)
date: 2017-11-04 19:25:14
tags: 
	- memory barriers
---


内存屏障是一种底层原语，在不同计算机架构下有不同的实现细节。本文主要在x86_64处理器下，通过Linux及其内核代码来分析和使用内存屏障

对大多数应用层开发者来说，“内存屏障”（memory barrier）是一种陌生，甚至有些诡异的技术。实际上，他常被用在操作系统内核中，用于实现同步机制、驱动程序等。利用它，能实现高效的无锁数据结构，提高多线程程序的性能表现。本文首先探讨了内存屏障的必要性，之后介绍如何使用内存屏障实现一个无锁唤醒缓冲区（队列），用于在多个线程间进行高效的数据交换。

<!-- more -->



# 理解内存屏障

不少开发者并不理解一个事实 -- 程序实际运行时很可能并不完全按照开发者编写的顺序访问内存。例如：

```cpp
x = r;
y = 1;
```

这里，y = 1很可能先于x = r执行。这就是内存乱序访问。内存乱序访问行为出现的理由是为了提升程序运行时的性能。编译器和CPU都可能引起内存乱序访问：

* 编译时，编译器优化进行指令重排而导致内存乱序访问；
* 运行时，多CPU间交互引入内存乱序访问。

编译器和CPU引入内存乱序访问通常不会带来什么问题，但在一些特殊情况下（主要是多线程程序中），逻辑的正确性依赖于内存访问顺序，这时，内存乱序访问会带来逻辑上的错误，例如：

```cpp
// thread 1
while(!ok);
do(x);

// thread 2
x = 42;
ok = 1;
```

`ok`初始化为0， 线程1等待`ok`被设置为1后执行do函数。假如，线程2对内存的写操作乱序执行，也就是`x`赋值晚于`ok`赋值完成，那么do函数接受的实参很有可能出乎开发者的意料，不为42。

我们可以引入内存屏障来避免上述问题的出现。内存屏障能让CPU或者编译器在内存访问上有序。一个内存屏障之前的内存访问操作必定先于其之后的完成。内存屏障包括两类：编译器屏障和CPU内存屏障。



# 编译时内存乱序访问

编译器对代码做出优化时，可能改变实际执行指令的顺序（例如g++下O2或者O3都会改变实际执行指令的顺序），看一个例子:

```cpp
int x, y, r;
void f()
{
    x = r;
    y = 1;
}
```

首先直接编译次源文件：`g++ -S test.cpp`。我们得到相关的汇编代码如下：

```asm
movl    r(%rip), %eax
movl    %eax, x(%rip)
movl    $1, y(%rip)
```

这里我们可以看到，`x = r`和`y = 1`并没有乱序执行。现使用优化选项O2(或O3)编译上面的代码（`g++ -O2 –S test.cpp`），生成汇编代码如下：

```asm
movl    r(%rip), %eax
movl    $1, y(%rip)
movl    %eax, x(%rip)
```

我们可以清楚地看到经过编译器优化之后，movl $1, y(%rip)先于movl %eax, x(%rip)执行，这意味着，编译器优化导致了内存乱序访问。避免次行为的办法就是使用编译器屏障（又叫优化屏障）。Linux内核提供了函数barrier()，用于让编译器保证其之前的内存访问先于其之后的内存访问完成。（这个强制保证顺序的需求在哪里？换句话说乱序会带来什么问题内？ -- 一个线程执行了 y =1 , 但实际上x=r还没有执行完成，此时被另一个线程抢占，另一个线程执行，发现y=1，以为此时x必定=r，执行相应逻辑，造成错误）内核实现barrier()如下：

```cpp
#define barrier() __asm__ __volatile__("": : :"memory")
```

现在把此编译器barrier加入代码中：

```cpp
int x, y, r;
void f()
{
x = r;
__asm__ __volatile__("": : :"memory")
    y = 1;
}
```

再编译，就会发现内存乱序访问已经不存在了。除了barrier()函数外，本例还可以使用volatile这个关键字来避免编译时内存乱序访问（且仅能避免编译时的乱序访问，为什么呢，可以参考前面部分的说明，编译器对于volatile声明究竟做了什么 – volatile关键字对于编译器而言，是开发者告诉编译器，这个变量内存的修改，可能不再你可视范围内，不要对这个变量相关的代码进行优化）。volatile关键字能让volatile变量之间的内存访问上有序，这里可以修改x和y的定义来解决问题：

```cpp
volatile int x, y, r;
```

通过volatile关键字，使得x相对y、y相对x在内存访问上是有序的。实际上，Linux内核中，宏`ACCESS_ONCE`能避免编译器对于连续的`ACCESS_ONCE`实例进行指令重排，其就是通过`volatile`实现的：

```cpp
#define ACCESS_ONCE(x) (*(volatile typeof(x) *)&(x))
```

此代码只是将变量x转换为volatile的而已。现在我们就有了第三个修改方案：

```cpp
int x, y, r;
void f()
{
	ACCESS_ONCE(x) = r;
    ACCESS_ONCE(y) = 1;
}
```

到此，基本上就阐述完成了编译时内存乱序访问的问题。下面看看CPU会有怎样的行为。

# 运行时内存乱序访问

运行时，CPU本身是会乱序执行指令的。早期的处理器为有序处理器（in-order processors）,总是按开发者编写的顺序执行指令，如果指令的输入操作对象（input operands）不可用（通常由于需要从内存中获取），那么处理器不会转而执行那些输入操作对象可用的指令，而是等待当前输入操作对象可用。相比之下，乱序处理器（out-of-order processors）会先处理那些有可用输入操作对象的指令（而非顺序执行）从而避免了等待，提高了效率。现代计算机上，处理器运行的速度比内存快很多，有序处理器花在等待可用数据的时间里已可处理大量指令了。即便现代处理器会乱序执行，但在单个CPU上，指令能通过指令队列顺序获取并执行，结果利用队列顺序返回寄存器堆（详情可参考http://en.wikipedia.org/wiki/Out-of-order_execution），这使得程序执行时所有的内存访问操作看起来像是按程序代码编写的顺序执行的，因此内存屏障是没有必要使用的（前提是不考虑编译器优化的情况下）。

SMP架构需要内存屏障的进一步解释：从体系结构上来看，首先在SMP架构下，每个CPU与内存之间，都配有自己的高速缓存（Cache），以减少访问内存时的冲突

![](http://og43lpuu1.bkt.clouddn.com/what_is_memory_barriers/img/01_smp_arch.jpg?imageView2/2/w/600/h/300)

采用高速缓存的写操作有两种模式：(1). 穿透(Write through)模式，每次写时，都直接将数据写回内存中，效率相对较低；(2). 回写(Write back)模式，写的时候先写回告诉缓存，然后由高速缓存的硬件再周转复用缓冲线(Cache Line)时自动将数据写回内存，或者由软件主动地“冲刷”有关的缓冲线(Cache Line)。出于性能的考虑，系统往往采用的是模式2来完成数据写入。正是由于存在高速缓存这一层，正是由于采用了Write back模式的数据写入，才导致在SMP架构下，对高速缓存的运用可能改变对内存操作的顺序。已上面的一个简短代码为例：

```cpp
// thread 0 -- 在CPU0上运行
x = 42;
ok = 1;

// thread 1 – 在CPU1上运行
while(!ok);
print(x);
```

这里CPU1执行时， x一定是打印出42吗？让我们来看看以下图为例的说明：

![](http://og43lpuu1.bkt.clouddn.com/what_is_memory_barriers/img/02_smp_cache_order_changed.jpg?imageView2/2/w/800/h/400)

假设，正好CPU0的高速缓存中有x，此时CPU0仅仅是将`x=42`写入到了高速缓存中，另外一个ok也在高速缓存中，但由于周转复用高速缓冲线（Cache Line）而导致将`ok=1`刷会到了内存中，此时CPU1首先执行对ok内存的读取操作，他读到了ok为1的结果，进而跳出循环，读取x的内容，而此时，由于实际写入的x(42)还只在CPU0的高速缓存中，导致CPU1读到的数据为x(17)。**程序中编排好的内存访问顺序（指令序： program ordering）是先写入x，再写入y。而实际上出现在该CPU外部，即系统总线上的次序（处理器序： processor ordering），却是先写入y，再写入x(这个例子中x还未写入)**。在SMP架构中，每个CPU都只知道自己何时会改变内存的内容，但是都不知道别的CPU会在什么时候改变内存的内容，也不知道自己本地的高速缓存中的内容是否与内存中的内容不一致。反过来，每个CPU都可能因为改变了内存内容，而使得其他CPU的高速缓存变的不一致了。在SMP架构下，由于高速缓存的存在而导致的内存访问次序（读或写都有可能书序被改变）的改变很有可能影响到CPU间的同步与互斥。因此需要有一种手段，使得在某些操作之前，把这种“欠下”的内存操作（本例中的x=42的内存写入）全都最终地、物理地完成，就好像把欠下的债都结清，然后再开始新的（通常是比较重要的）活动一样。这种手段就是内存屏障，其本质原理就是对系统总线加锁。

回过头来，我们再来看看为什么非SMP架构（UP架构）下，运行时内存乱序访问不存在。在单处理器架构下，各个进程在宏观上是并行的，但是在微观上却是串行的，因为在同一时间点上，只有一个进程真正在运行（系统中只有一个处理器）。在这种情况下，我们再来看看上面提到的例子：

![](http://og43lpuu1.bkt.clouddn.com/what_is_memory_barriers/img/03_up_cache_order_unchanged.jpg?imageView2/2/w/500/h/400)

线程0和线程1的指令都将在CPU0上按照指令序执行。thread0通过CPU0完成`x=42`的高速缓存写入后，再将`ok=1`写入内存，此后串行的将thread0换出，thread1换入，及时此时`x=42`并未写入内存，但由于thread1的执行仍然是在CPU0上执行，他仍然访问的是CPU0的高速缓存，因此，及时`x=42`还未写回到内存中，thread1势必还是先从高速缓存中读到`x=42`，再从内存中读到`ok=1`

综上所述，在单CPU上，多线程执行不存在运行时内存乱序访问，我们从内核源码也可得到类似结论（代码不完全摘录）

```cpp
#define barrier() __asm__ __volatile__("": : :"memory") 
#define mb() alternative("lock; addl $0,0(%%esp)", "mfence", X86_FEATURE_XMM2) 
#define rmb() alternative("lock; addl $0,0(%%esp)", "lfence", X86_FEATURE_XMM2)

#ifdef CONFIG_SMP 
#define smp_mb() mb() 
#define smp_rmb() rmb() 
#define smp_wmb() wmb() 
#define smp_read_barrier_depends() read_barrier_depends() 
#define set_mb(var, value) do { (void) xchg(&var, value); } while (0) 
#else 
#define smp_mb() barrier() 
#define smp_rmb() barrier() 
#define smp_wmb() barrier() 
#define smp_read_barrier_depends() do { } while(0) 
#define set_mb(var, value) do { var = value; barrier(); } while (0) 
#endif
```

这里可看到对内存屏障的定义，如果是SMP架构，smp_mb定义为mb()，mb()为CPU内存屏障（接下来要谈的），而非SMP架构时（也就是UP架构），直接使用编译器屏障，运行时内存乱序访问并不存在。

为什么多CPU情况下会存在内存乱序访问？我们知道每个CPU都存在Cache，当一个特定数据第一次被其他CPU获取时，此数据显然不在对应CPU的Cache中（这就是Cache Miss）。这意味着CPU要从内存中获取数据（这个过程需要CPU等待数百个周期），此数据将被加载到CPU的Cache中，这样后续就能直接从Cache上快速访问。当某个CPU进行写操作时，他必须确保其他CPU已将此数据从他们的Cache中移除（以便保证一致性），只有在移除操作完成后，此CPU才能安全地修改数据。显然，存在多个Cache时，必须通过一个Cache一致性协议来避免数据不一致的问题，而这个通信的过程就可能导致乱序访问的出现，也就是运行时内存乱序访问。受篇幅所限，这里不再深入讨论整个细节，有兴趣的读者可以研究《Memory Barriers: a Hardware View for Software Hackers》这边文章，它详细地分析了整个过程。

现在通过一个例子来直观地说明多CPU下内存乱序访问的问题：

```cpp
volatile int x, y, r1, r2;
//thread 1
void run1()
{
    x = 1;
    r1 = y;
}

//thread 2
void run2
{
    y = 1;
    r2 = x;
}
```

变量`x、y、r1、r2`均被初始化为0，run1和run2运行在不同的线程中。如果run1和run2在同一个cpu下执行完成，那么就如我们所料，r1和r2的值不会同时为0，而假如run1和run2在不同的CPU下执行完成后，由于存在内存乱序访问的可能，这时r1和r2可能同时为0。我们可以使用CPU内存屏障来避免运行时内存乱序访问(x86_64)：

```cpp
void run1()
{
    x = 1;
    //CPU内存屏障，保证x=1在r1=y之前执行
    __asm__ __volatile__("mfence":::"memory");
    r1 = y;
}

//thread 2
void run2
{
    y = 1;
    //CPU内存屏障，保证y = 1在r2 = x之前执行
    __asm__ __volatile__("mfence":::"memory");
    r2 = x;
}
```

这里mfence的含义是什么？
x86/64系统架构提供了三中内存屏障指令：(1) sfence; (2) lfence; (3) mfence。（参考介绍：http://peeterjoot.wordpress.com/2009/12/04/intel-memory-ordering-fence-instructions-and-atomic-operations/ 以及Intel文档：http://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-3a-part-1-manual.pdf 和http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html）

1. sfence
Performs a serializing operation on all store-to-memory instructions that were issued prior the SFENCE instruction. **This serializing operation guarantees that every store instruction that precedes in program order the SFENCE instruction is globally visible before any store instruction that follows the SFENCE instruction is globally visible**. The SFENCE instruction is ordered with respect store instructions, other SFENCE instructions, any MFENCE instructions, and any serializing instructions (such as the CPUID instruction). It is not ordered with respect to load instructions or the LFENCE instruction.

	也就是说sfence确保：sfence指令前后的写入(store/release)指令，按照在sfence前后的指令序进行执行。写内存屏障提供这样的保证: 所有出现在屏障之前的STORE操作都将先于所有出现在屏障之后的STORE操作被系统中的其他组件所感知.
    
	[!] 注意, 写屏障一般需要与读屏障或数据依赖屏障配对使用; 参阅"SMP内存屏障配对"章节. (译注: 因为写屏障只保证自己提交的顺序, 而无法干预其他代码读内存的顺序. 所以配对使用很重要. 其他类型的屏障亦是同理.)
    
2. lfence
Performs a serializing operation on all load-from-memory instructions that were issued prior the LFENCE instruction. **This serializing operation guarantees that every load instruction that precedes in program order the LFENCE instruction is globally visible before any load instruction that follows the LFENCE instruction is globally visible**. The LFENCE instruction is ordered with respect to load instructions, other LFENCE instructions, any MFENCE instructions, and any serializing instructions (such as the CPUID instruction). It is not ordered with respect to store instructions or the SFENCE instruction.

	也就是说lfence确保：lfence指令前后的读取(load/acquire)指令，按照在mfence前后的指令序进行执行。读屏障包含数据依赖屏障的功能, 并且保证所有出现在屏障之前的LOAD操作都将先于所有出现在屏障之后的LOAD操作被系统中的其他组件所感知.

	[!] 注意, 读屏障一般要跟写屏障配对使用; 参阅"SMP内存屏障的配对使用"章节.

3. mfence
Performs a serializing operation on all load-from-memory and store-to-memory instructions that were issued prior the MFENCE instruction. **This serializing operation guarantees that every load and store instruction that precedes in program order the MFENCE instruction is globally visible before any load or store instruction that follows the MFENCE instruction is globally visible**. The MFENCE instruction is ordered with respect to all load and store instructions, other MFENCE instructions, any SFENCE and LFENCE instructions, and any serializing instructions (such as the CPUID instruction).

	也就是说mfence指令：确保所有mfence指令之前的写入(store/release)指令，都在该mfence指令之后的写入（store/release）指令之前（指令序，Program Order）执行；同时，他还确保所有mfence指令之后的读取（load/acquire）指令，都在该mfence指令之前的读取（load/acquire）指令之后执行。即：既确保写者能够按照指令序完成数据写入，也确保读者能够按照指令序完成数据读取。通用内存屏障保证所有出现在屏障之前的LOAD和STORE操作都将先于所有出现在屏障之后的LOAD和STORE操作被系统中的其他组件所感知.
    
sfence我认为其动作，可以看做是一定将数据写回内存，而不是写到高速缓存中。lfence的动作，可以看做是一定将数据从高速缓存中抹掉，从内存中读出来，而不是直接从高速缓存中读出来。mfence则正好结合了两项操作。sfence只确保写者在将数据（A->B）写入内存的顺序，并不确保其他人读(A,B)数据时，一定是按照先读A更新后的数据，再读B更新后的数据这样的顺序，很有可能读者读到的顺序是A旧数据，B更新后的数据，A更新后的数据（只是这个更新后的数据出现在读者的后面，他并没有“实际”去读）；同理，lfence也就只能确保读者在读入顺序时，按照先读A最新的在内存中的数据，再读B最新的在内存中的数据的顺序，但如果没有写者sfence的配合，显然，即使顺序一致，内容还是有可能乱序。

为什么仅通过保证了写者的写入顺序(sfence), 还是有可能有问题？还是之前的例子

```cpp
void run1()
{
    x = 1;
    //CPU内存屏障，保证x=1在r1=y之前执行
    __asm__ __volatile__("sfence":::"memory");
    r1 = y;
}

//thread 2
void run2
{
    y = 1;
    //CPU内存屏障，保证y = 1在r2 = x之前执行
    __asm__ __volatile__("sfence":::"memory");
    r2 = x;
}
```

如果仅仅是对“写入”操作进行顺序化，实际上，还是有可能使的上面的代码出现r1，r2同时为0（初始值）的场景：

![](http://og43lpuu1.bkt.clouddn.com/what_is_memory_barriers/img/04_sfence_limit.jpg?imageView2/2/w/800/h/400)

当CPU0上的thread0执行时，x被先行写回到内存中，但如果此时y在CPU0的高速缓存中，这时y从缓存中读出，并被赋予r1写回内存，此时r1为0。同理，CPU1上的thread1执行时，y被先行写回到内存中，如果此时x在CPU1的高速缓存中存在，则此时r2被赋予了x的（过时）值0，同样存在了r1, r2同时为0。这个现象实际上就是所谓的`r1=y`的读顺序与`x=1`的写顺序存在逻辑上的乱序所致（或者是`r2 = x`与`y=1`存在乱序） -- 读操作与写操作之间存在乱序。而mfence就是将这类乱序也屏蔽掉

如果是通过mfence，是怎样解决该问题的呢？

![](http://og43lpuu1.bkt.clouddn.com/what_is_memory_barriers/img/05_mfence_usage.jpg?imageView2/2/w/800/h/400)

当thread1在CPU0上对`x=1`进行写入时，`x=1`被刷新到内存中，由于是mfence，他要求r1的读取操作从内存读取数据，而不是从缓存中读取数据，因此，此时如果y更新为1，则`r1 = 1`; 如果y没有更新为1，则`r1 = 0`， 同时此时由于x更新为1， r2必须从内存中读取数据，则此时`r2 = 1`。总而言之是r1, r2, 一个=0， 一个=1。

# 关于内存屏障的一些补充

在实际的应用程序开发中，开发者可能完全不知道内存屏障就写出了正确的多线程程序，这主要是因为各种同步机制中已隐含了内存屏障（但和实际的内存屏障有细微差别），使得不直接使用内存屏障也不会存在任何问题。但如果你希望编写诸如无锁数据结构，那么内存屏障意义重大。

在Linux内核中，除了前面说到的编译器屏障—barrier()和ACESS_ONCE()，还有CPU内存屏障：

* 通用屏障，保证读写操作有序，包括mb()和smp_mb();
* 写操作屏障，仅保证写操作有序，包括wmb()和smp_wmb();
* 读操作屏障，仅保证读操作有序，包括rmb()和smp_rmb();

注意，所有的CPU内存屏障（除了数据依赖屏障外）都隐含了编译器屏障（也就是使用CPU内存屏障后就无需再额外添加编译器屏障了）。这里的smp开通的内存屏障会根据配置在单处理器上直接使用编译器屏障，而在SMP上才使用CPU内存屏障（即mb()、wmb()、rmb()）。

还需要注意一点是，CPU内存屏障中某些类型的屏障需要成对使用，否则会出错，详细来说就是：一个写操作屏障需要和读操作（或者数据依赖）屏障一起使用（当然，通用屏障也是可以的），反之亦然。

通常，我们是希望在写屏障之前出现的STORE操作总是匹配度屏障或者数据依赖屏障之后出现的LOAD操作。以之前的代码示例为例：

```cpp
// thread 1
x = 42;
smb_wmb();
ok = 1;

// thread 2
while(!ok);
smb_rmb();
do(x);
```

我们这么做，是希望在thread2执行到do(x)时（在ok验证的确=1时），x = 42的确是有效的(写屏障之前出现的STORE操作)，此时do(x)，的确是在执行do(42)（读屏障之后出现的LOAD操作）

## 利用内存屏障实现无锁环形缓冲区

最后，以一个使用内存屏障实现的无锁环形缓冲区（只有一个读线程和一个写线程时）来结束本文。本代码源于内核FIFO的一个实现，内容如下（略去了非关键代码）：
*代码来源：linux-2.6.32.63\kernel\kfifo.c*

```cpp
unsigned int __kfifo_put(struct kfifo *fifo,
			const unsigned char *buffer, unsigned int len)
{
	unsigned int l;

	len = min(len, fifo->size - fifo->in + fifo->out);

	/*
	 * Ensure that we sample the fifo->out index -before- we
	 * start putting bytes into the kfifo.
     * 通过内存屏障确保先读取fifo->out后，才将buffer中数据拷贝到
     * 当前fifo中
	 */

	smp_mb();

	/* first put the data starting from fifo->in to buffer end */
    /* 将数据拷贝至fifo->in到fifo结尾的一段内存中 */
	l = min(len, fifo->size - (fifo->in & (fifo->size - 1)));
	memcpy(fifo->buffer + (fifo->in & (fifo->size - 1)), buffer, l);

	/* then put the rest (if any) at the beginning of the buffer */
    /* 如果存在剩余未拷贝完的数据（此时len – l > 0）则拷贝至 
    * fifo的开始部分
    */
	memcpy(fifo->buffer, buffer + l, len - l);

	/*
	 * Ensure that we add the bytes to the kfifo -before-
	 * we update the fifo->in index.
	 */
     
    /*
     * 通过写操作屏障确保数据拷贝完成后才更新fifo->in
     */

	smp_wmb();

	fifo->in += len;

	return len;
}
```

```cpp
unsigned int __kfifo_get(struct kfifo *fifo,
			 unsigned char *buffer, unsigned int len)
{
	unsigned int l;

	len = min(len, fifo->in - fifo->out);

	/*
	 * Ensure that we sample the fifo->in index -before- we
	 * start removing bytes from the kfifo.
	 */
    /*
     * 通过读操作屏障确保先读取fifo->in后，才执行另一个读操作：
     * 将fifo中数据拷贝到buffer中去
     */

	smp_rmb();

	/* first get the data from fifo->out until the end of the buffer */
    /* 从fifo->out开始拷贝数据到buffer中 */
	l = min(len, fifo->size - (fifo->out & (fifo->size - 1)));
	memcpy(buffer, fifo->buffer + (fifo->out & (fifo->size - 1)), l);

	/* then get the rest (if any) from the beginning of the buffer */
    /* 如果需要数据长度大于fifo->out到fifo结尾长度，
     * 则从fifo开始部分拷贝（此时len – l > 0）
     */
	memcpy(buffer + l, fifo->buffer, len - l);

	/*
	 * Ensure that we remove the bytes from the kfifo -before-
	 * we update the fifo->out index.
	 */
    /* 通过内存屏障确保数据拷贝完后，才更新fifo->out */

	smp_mb();

	fifo->out += len;

	return len;
}

```

这里`__kfifo_put`被一个线程用于向`fifo`中写入数据，另外一个线程可以安全地调用`__kfifo_get`，从此`fifo`中读取数据。代码中in和out的索引用于指定环形缓冲区实际的头和尾。具体的in和out所指向的缓冲区的位置通过与操作来求取（例如：`fifo->in & (fifo->size -1)`），这样相比取余操作来求取下表的做法效率要高不少。使用与操作求取下表的前提是环形缓冲区的大小必须是2的N次方，换而言之，就是说环形缓冲区的大小为一个仅有一个1的二进制数，那么`index & (size – 1)`则为求取的下标（这不难理解）。

![](http://og43lpuu1.bkt.clouddn.com/what_is_memory_barriers/img/06_kfifo.jpg?imageView2/2/w/700/h/300)

索引in和out被两个线程访问。in和out指明了缓冲区中实际数据的边界，也就是in和out同缓冲区数据存在访问上的顺序关系，由于不适用同步机制，那么保证顺序关系就需要使用到内存屏障了。索引in和out都分别只被一个线程修改，而被两个线程读取。`__kfifo_put`先通过in和out来确定可以向缓冲区中写入数据量的多少，这时，out索引应该先被读取，才能真正将用户buffer中的数据写入缓冲区，因此这里使用到了`smp_mb()`，对应的，`__kfifo_get`也使用`smp_mb()`来确保修改out索引之前缓冲区中数据已被成功读取并写入用户buffer中了。（我认为在`__kfifo_put`中添加的这个`smp_mb()`是没有必要的。理由如下，kfifo仅支持一写一读，这是前提。在这个前提下，in和out两个变量是有着依赖关系的，这的确没错，并且我们可以看到在put中，in一定会是最新的，因为put修改的是in的值，而在get中，out一定会是最新的，因为get修改out的值。这里的smp_mb()显然是希望在运行时，遵循out先load新值，in再load新值。的确，这样做是没错，但这是否有必要呢？out一定要是最新值吗？out如果不是最新值会有什么问题？如果out不是最新值，实际上并不会有什么问题，仅仅是在put时，fifo的实际可写入空间要大于put计算出来的空间（因为out是旧值，导致len在计算时偏小），这并不影响程序执行的正确性。从最新linux-3.16-rc3 kernel的代码：lib\kfifo.c的实现：`__kfifo_in`中也可以看出`memcpy(fifo->data + off, src, l); memcpy(fifo->data, src + l, len - l);`之前的那次`smb_mb()`已经被省去了，当然更新in之前的`smb_wmb()`还是在`kfifo_copy_in`中被保留了。之所以省去这次`smb_mb()`的调用，我想除了省去调用不影响程序正确性外，是否还有对于性能影响的考虑，尽量减少不必要的mb调用）对于in索引，在`__kfifo_put`中，通过`smp_wmb()`保证先向缓冲区写入数据后才修改in索引，由于这里只需要保证写入操作有序，所以选用写操作屏障，在`__kfifo_get`中，通过`smp_rmb()`保证先读取了in索引（这时in索引用于确定缓冲区中实际存在多少可读数据）才开始读取缓冲区中数据（并写入用户buffer中），由于这里指需要保证读取操作有序，故选用读操作屏障。

## 什么时候需要注意考虑内存屏障(补充)

从上面的介绍我们已经可以看出，在SMP环境下，内存屏障是如此的重要，在多线程并发执行的程序中，一个数据读取与写入的乱序访问，就有可能导致逻辑上错误，而显然这不是我们希望看到的。作为系统程序的实现者，我们涉及到内存屏障的场景主要集中在无锁编程时的原子操作。执行这些操作的地方，就是我们需要考虑内存屏障的地方。

从我自己的经验来看，使用原子操作，一般有如下三种方式：(1). 直接对int32、int64进行赋值；(2). 使用gcc内建的原子操作内存访问接口；(3). 调用第三方atomic库：libatomic实现内存原子操作。

1. 对于第一类原子操作方式，显然内存屏障是需要我们考虑的，例如kernel中kfifo的实现，就必须要显示的考虑在数据写入和读取时插入必要的内存屏障，以保证程序执行的顺序与我们设定的顺序一致。
2. 对于使用gcc内建的原子操作访问接口，基本上大多数gcc内建的原子操作都自带内存屏障，他可以确保在执行原子内存访问相关的操作时，执行顺序不被打断。“In most cases, these builtins are considered a full barrier. That is, no memory operand will be moved across the operation, either forward or backward. Further, instructions will be issued as necessary to prevent the processor from speculating loads across the operation and from queuing stores after the operation.”（http://gcc.gnu.org/onlinedocs/gcc-4.4.5/gcc/Atomic-Builtins.html）。当然，其中也有几个并未实现full barrier，具体情况可以参考gcc文档对对应接口的说明。同时，gcc还提供了对内存屏障的封装接口：__sync_synchronize (...)，这可以作为应用程序使用内存屏障的接口(不用写汇编语句)。
3. 对于使用libatomic库进行原子操作，原子访问的程序。Libatomic在接口上对于内存屏障的设置粒度更新，他几乎是对每一个原子操作的接口针对不同平台都有对应的不同内存屏障的绑定。“Provides implementations for atomic memory update operations on a number of architectures. This allows direct use of these in reasonably portable code. Unlike earlier similar packages, this one explicitly considers memory barrier semantics, and allows the construction of code that involves minimum overhead across a variety of architectures.”接口实现上分别添加了_release/_acquire/_full等各个后缀，分别代表的该接口的内存屏障类型，具体说明可参见libatomic的README说明。如果是调用最顶层的接口，已AO_compare_and_swap为例，最终会根据平台的特性以及宏定义情况调用到：AO_compare_and_swap_full或者AO_compare_and_swap_release或者AO_compare_and_swap_release等。我们可以重点关注libatomic在x86_64上的实现，libatomic中，在x86_64架构下，也提供了应用层的内存屏障接口：AO_nop_full

综合上述三点，总结下来便是：如果你在程序中是裸着写内存，读内存，则需要显式的使用内存屏障确保你程序的正确性，gcc内建不提供简单的封装了内存屏障的内存读写，因此，如果只是使用gcc内建函数，你仍然存在裸读，裸写，此时你还是必须显式使用内存屏障。如果你通过libatomic进行内存访问，在x86_64架构下，使用AO_load/AO_store，你可以不再显式的使用内存屏障（但从实际使用的情况来看，libatomic这类接口的效率不是很高）