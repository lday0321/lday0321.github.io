---
title: Linux下Condition Vairable和Mutext合用的小细节
date: 2017-11-19 14:04:14
tags: 
	- linux
	- concurrency
---

在Linux下合用Condition Variable和Mutex有些小细节。为什么需要这么做，总结于此。

<!-- more -->

mutex用于上锁，condition variable用于等待，这两种不同类型的同步，都是我们所需要的。对于条件变量，它提供的操作接口主要为：

```c
#include <pthread.h>

pthread_cond_t cond = PTHREAD_COND_INITIALIZER;

int pthread_cond_signal(pthread_cond_t *cond);
int pthread_cond_broadcast(pthread_cond_t *cond);
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);

// All return 0 on success, or a positive error number on error
```

pthread_cond_signal的作用是通知那些wait在cond上的线程，有事件达到。而pthread_cond_wait的作用实际上包含3部分：

1. 将与之绑定的mutex解锁(unlock)
2. 让自己进入等待状态，直至被signal唤醒
3. 在被唤醒之后再次拿到与之绑定的mutex

从pthread_cond_wait返回，实际上是他重新又拿到了mutex

基于mutex + cond的生产者往往如下所示：
```c
s = pthread_mutex_lock(&mtx);			//加锁，执行生产动作
if (s != 0)
	errExitEN(s, "pthread_mutex_lock");

avail++;  							//实际的生产相应数据

s = pthread_mutex_unlock(&mtx);		//完成生产之后，首先将mutex释放
if (s != 0)
	errExitEN(s, "pthread_mutex_unlock");


s = pthread_cond_signal(&cond); 		//之后，再通过signal，通知消费者
if (s != 0)
	errExitEN(s, "pthread_cond_signal");
```

先unlock，再执行signal，这种操作顺序，我认为是比较符合习惯的。但在《Unix网络编程，卷2》的7.5章节中，我们看到了另外一种写法，先signal，再unlock。

![](http://og43lpuu1.bkt.clouddn.com/condition_variable_and_mutex_together/img01_condidition_variable_producer_seq.png)

**我直观的认为这样不合适，毕竟signal之后，那些在等待的线程就唤醒，而此时如果没法拿到锁，就会进入竞争，而且是不是有可能这个时候没有一个人拿到锁，都又睡去呢？**（因为生产者mutex还没能unlock时，所有消费者都拿不到锁，都又睡去了，有没有可能？）-- 虽然这种最终没有消费者唤醒的情况不会存在，但的确还是有些影响，根据《The Linux Programming Interface》中的描述：

> on some implementations, <font color="#FF0000">unlocking the mutex and then signaling the condition variable may yield better performance than performing these steps in the reverse sequence.</font> If the mutex is unlocked only after the condition variable is signaled, the thread performing pthread_cond_wait() may wake up while the mutex is still locked, and then immediately go back to sleep again when it finds that the mutex is locked. <font color="#FF0000">This results in two superfluous context switches.</font> Some implementations eliminate this problem by employing a technique called <font color="#0000FF">**wait morphing**</font>, which moves the signaled thread from the condition variable wait queue to the mutex wait queue without performing a context switch if the mutex is locked.

所以，还是按照上面，先放锁，再signal会比较好。

看完生产者的实现，我们再看看消费者的实现：

```c
for (;;) {
	//消费者首先加锁查看资源情况
	s = pthread_mutex_lock(&mtx);
	if (s != 0)
		errExitEN(s, "pthread_mutex_lock");
	
	while (avail == 0) { 	//循环等待有可用资源进行消费，
		//在没有可用资源进行消费时，就放锁，进入休眠
		s = pthread_cond_wait(&cond, &mtx);		
		if (s != 0)
			errExitEN(s, "pthread_cond_wait");
	}
	
	//循环等待出来之后，是实际上有资源可消费，同时自己持有了mutex
	while (avail > 0) { 	//进行消费
		//扣减资源，进行消费
		avail--;
	}
	
	//完成消费后，释放mutex
	s = pthread_mutex_unlock(&mtx);
	if (s != 0)
		errExitEN(s, "pthread_mutex_unlock");
	
}
```

这里有一点需要注意的是：<font color="#0000FF">pthread_cond_wait(…)是被包裹在一个while(…)判断语句中的，判断是否有资源，并不是用的if，而是用的while。</font>之所以用while，**是因为即使在被pthread_cond_wait(…)唤醒，拿到了mutex，也存在实际上并没有资源的情况**。

1. 其他线程先于你被唤醒，并将资源消费完了，你才被唤醒。“pthread_cond_signal(), we are simply guaranteed that **at least** one of the blocked threads is woken up;”从这里就可以看出，也许有多个线程被唤醒，并以此拿到了mutex，因为前面拿到mutex的线程可能已经消费完了资源，此时你被唤醒，实际上是没有用处了，此时就应该再读进入wait，因此需要通过while()而不是if()来控制是否能够进入真正的消费阶段

2. 可能发生惊群。在一些pthread_cond_wait(…)的实现中，即使没有人调用signal，一个线程也有可能在pthread_cond_wait(…)上被唤醒，这种惊群现象是一些多核处理器技术实现的后果，而这种惊群现象在SUSv3中是被显式允许的，因此我们不能假设从wait中醒来就一定有资源可消费，还必须再探测一下。
