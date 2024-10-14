---
title: 生产者｜参数配置
categories:
  - mq
tags:
  - mq
  - kafka
abbrlink: 8c16cb10
date: 2023-12-09 23:00:00
---

> 本系列内容基于 kafka_2.12-2.3.1

headers 字段是消息的头部， Kafka 0.11 版本才引入这个属性。它大多用来设定一些与应用相关的信息，如无
需要也可以不用设置。

value 是指消息体， 一般不为空，如果为空则表示特定的消息一一墓碑消息。

timestamp 是指消息的时间戳，它有 CreateTime LogAppendTime 两种类型，前者表示消息创建的时间，后者表示消息追加到
日志文件的时间。

<!-- more -->

# bootstrap.servers

该参数用来指定生产者客户端连接 Kafka 集群所需的 broker 地址清单，具体的内容格式为 hostl:portl,hos t2:port2。

# client.id

用来设定 KafkaProducer 应的客户端 id 默认值为"" 如果客户端不设置， KafkaProducer自动生成一个非空字符串，内容形式如"producer-1" "producer-2"，即字符串“producer-"与数字的拼接。

KafkaProducer 是线程安全的，可以在多个线程中共享单个 KafkaProducer 实例，也可以将 KafkaProducer 实例进行池化来供其他线程调用。

# acks

这个参数用来指定分区中必须要有多少个副本收到这条消息，之后生产者才会认为这条消息是成功写入的。 

acks 是生产者客户端中一个非常重要 参数 ，它涉及消息的可靠性和吞吐量之间的权衡 acks 参数有 种类型的值（都是字符串类型）。

- 0

生产者发送消息之后不需要等待任何服务端的响应。

- 1
。生产者发送消息之后，只要分区的 eader 副本成功写入消息，那么它就会收到来自服务端的成功响应。

- -1

生产者在消 息发送之后，需要等待 ISR 中的所有副本都成功写入消息之后才能够收到来自服务端的成功响应。

一般而言，在需要保证消息顺序的场合建议把参数 max.in.flight.requests.per.connection 配置为 1，而不是把 acks 配置为 0
不过这样也会影响整体的吞吐。

# max.equest.size

用来限制生产者客户端能发送的消息的最大值，默认值为 1048576B ，即 1MB。不建议盲目增大它，因为它和其他参数有联动效应：比如和 broker端的 message.max.bytes。 如果当前参数比 message.max.bytes大，就会有问题。

# retries 和 retry.backoff.ms

retries 参数用来配置生产者重试的次数，默认值为0，即在发生异常的时候不进行任何重试动作。

retry.backoff.ms 它用来设定两次重试之间的时间间隔，避免无效的频繁重试。默认为100

# compression.type

这个参数用来指定消息的压缩方式，默认值为 "none"，即默认情况下，消息不会被压缩。如果对时延有一定的要求，则不推荐对消息进行压缩。
