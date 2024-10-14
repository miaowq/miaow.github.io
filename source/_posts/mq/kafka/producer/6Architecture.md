---
title: 生产者｜整体架构
categories:
  - mq
tags:
  - mq
  - kafka
abbrlink: 8d90bad2
date: 2023-12-09 23:00:05
---

> 本系列内容基于 kafka_2.12-2.3.1

![](/images/mq/kafka/kafka24.png)

<!-- more -->

# 2个线程

整个生产者客户端由两个线程协调运行，这两个线程分别为主线程和 Sender 线程 （发送线程）。

在主线程中由 KafkaProducer 创建消息，然后通过可能的拦截器、序列化器和分区器的作用之后缓存到消息累加器（ RecordAccumulator ，也称为消息收集器〉中。 Sender 线程负责从RecordAccumulator 获取消息并将其发送到 Kafka中。

# 消息累加器
RecordAccumulator 主要用来缓存消息 Sender 线程可以批量发送，进而减少网络传输的资源消耗以提升性能 RecordAccumulator 缓存的大小可以通过生产者客户端参数 buffer.memory 配置，默认值为 33554432B ，即 32M。

如果生产者发送消息的速度超过发送到服务器的速度 ，则会导致生产者空间不足，这个时候 KafkaProducer的send（）方法调用要么
被阻塞，要么抛出异常，这个取决于参数 max.block.ms 的配置，此参数的默认值为 60000，即60秒。

主线程中发送过来的消息都会被迫加到 RecordAccumulator 的某个双端队列（ Deque ）中。消息写入缓存时，追加到双端队列的尾部。Sender读取消息时 ，从双端队列的头部读取。

## BufferPool

消息在网络上都是以字节（Byte）的形式传输的，在发送之前需要创建 块内存区域来保存对应的消息。在 Kafka 生产者客户端中，通过 java.io.ByteBuffer 实现消息内存的创建和释放。不过频繁的 建和释放是比较耗费资源的。

它主要用来实现 ByteBuffer 的复用，以实现缓存的高效利用。

不过 BufferPool 只针对特定大小的 ByteBuffer 进行管理，而其他大小的 ByteBuffer 不会缓存进 BufferrPool 中。

这个特定的大小 batch.size 参数来指定，默认值为 16384B ，即 16KB 我们可以适当地调大 batch.size参数以便多缓存一些消息。

# ProducerBatch

可以包含一至多个 ProducerRecord。通俗地说，ProducerRecord 是生产者中创建的消息，而ProducerBatch 是指一个消息批次 ProducerRecord 会被包含在 ProducerBatch 中，这样可以使字节的使用更加紧凑。与此同时，将较小的 ProducerRecord 凑成一个较大 ProducerBatch ，也可以减少网络请求的次数以提升整体的吞吐量。

