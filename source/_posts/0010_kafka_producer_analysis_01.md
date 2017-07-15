---
title: Kafka Procuder小结
date: 2017-07-15 14:05:05
tags: 
	- Kafka
---

理解Kafka 0.10.0.1版本的Procuder代码，对Producer的整体逻辑进行小结。

<!-- more -->

# 1. 线程结构

![](http://og43lpuu1.bkt.clouddn.com/kafka_producer_analysis_01/png/01_producer_thread_structure.png)

如上图所示，Producer的功能主要由两个线程协同工作完成：
1. 主线程主要将业务数据封装成ProducerRecord对象，压入由RecordAccumulator构建的数据Queue中
2. Sender线程主要从Queue中获取数据，封装成请求报文，并最终通过网络IO线程将数据发送到Kafka

# 2. 模块结构

![](http://og43lpuu1.bkt.clouddn.com/kafka_producer_analysis_01/png/02_producer_modules_structure.png)



# 3. 交互逻辑

上图所示为Producer内部的模块关系图，整个消息发送流程大致分为如下步骤：
1. 用户调用send接口进入KafkaProducer内部处理逻辑，消息交由ProducerInterceptors对消息进行拦截处理
2. Serializer对消息的Key, Value进行序列化处理
3. Partitioner根据一定的策略为消息选择合适的分区
4. 消息被append到RecordAccumulator中，等待被发送线程获取
5. Sender从RecordAccumulator中获取消息，
6. 消息被封装成ClientRequest
7. ClientRequest被交给NetworkClient，准备执行消息发送
8. Network将请求放入KafkaChannel缓存中
9. 通过网络IO完成消息发送
10. 收到响应，调用ClientRequest上的异步回调
11. 对应的调用RecordBatch上的回调，最终调用每个消息上注册的回调函数


