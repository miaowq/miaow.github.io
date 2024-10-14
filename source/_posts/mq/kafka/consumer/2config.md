---
title: 消费者｜参数配置
categories:
  - mq
tags:
  - mq
  - kafka
abbrlink: 87a26e71
date: 2023-12-08 23:00:01
---

> 本系列内容基于 kafka_2.12-2.3.1
> 
> Golang Java

# 必要的参数配置

# bootstrap.servers

该参数的释义和生产者客户端 KafkaProducer 中的相同，用来指定连接 Kafka 集群所需的 broker 地址清单，具体内容形式为 host1:port1,host2:post，可以设置一个或多个地址，中间用逗号隔开，此参数的默认值为""

<!-- more -->

注意这里并非需要设置集群中全部的 broker 地址，消费者会从现有的配置中查找到全部的 Kafka 集群成员。这里设置两个以上的 broker 地址信息，当其中任意一个宕机时，消费者仍然可以连接到 Kafka 集群上。

# group.id

消费者隶属的消费组的名称，默认值为""

一般而言，这个参数需要设置成具有一定的业务意义的名称。

# 非必需 client.id

设定 KafkaConsumer 对应的客户端id，默认值为""

如果客户端不设置，则 KafkaConsumer 会自动生成一个非空字符串，内容形式如“consumer-1”、“consumer-2”，即字符串“consumer-”与数字的拼接。

# heartbeat.interval.ms
消费者的心跳间隔。默认为 3000，也就是3s。只要消费者以正常的时间间隔发送就被认为是活跃的，说明它还在取分区中的消息。**心跳线程是一个独立的线程，可以在轮询消息的空档发送心跳。**如果消费者停止发送心跳的时间足够长，则整个会话就判定过期，GPC认为这个消费者死亡，就会触发一次再平衡。**这个参数必须比 session.timeout.ms 参数设定的值要小**一般情况下 `heartbeat.interval.ms` 的配置值不能超过 `session.timeout.ms` 配置值的1/3 。

# session.timeout.ms
如果一个消费者发生崩溃，并停止读取消息，那么 GPC 会等待一段时间，确认这个消费者死亡之后才会触发再均衡。在这一小段时间内， 死掉的消费者井不会读取分区里的消息。这个一小段时间由 `session.timeout.ms` 参数控制。该参数的配置值必须在 `group.min.session.timeout.ms` 和 `group.max.session.timeout.ms` 允许的范围内。

# group.min.session.timeout.ms

# group.max.session.timeout.ms

# max.poll.interval.ms