---
title: 生产者｜序列化
categories:
  - mq
tags:
  - mq
  - kafka
abbrlink: 3af04756
date: 2023-12-09 23:00:02
---

> 本系列内容基于 kafka_2.12-2.3.1

生产者需要 列化器（ Seriali er ）把对象转换成字节数组才能通过网络发送给 Kafk 。而在对侧，消费者需要用反序列化器（ Deserializer ）把从 Kafka 中收到的字节数组转换成相应对象。

生产者使用的序列化器和消费者使用的反序列化器是需要一一对应的。