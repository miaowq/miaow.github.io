---
title: 生产者｜分区器
categories:
  - mq
tags:
  - mq
  - kafka
abbrlink: 37cfd85e
date: 2023-12-09 23:00:03
---

> 本系列内容基于 kafka_2.12-2.3.1

消息在通过 send（）方法发往 brokr 中， 有可能需要经过拦截器（ Interceptor ）、序列器（ Serializer ）和分区器（ Partition ）的一系列作用之后才能被真正地发 broker 。

如果消息中没有指定partition 字段，那么就需要依赖分区器 根据 key 这个字段来 partition 值。分区器的作用就是为消息分配分区。如果key 为null，那么消息将会以轮询的方式发往主题内的各个可用分区。

在不改变主题分区数量的情况下 key 与分区之间的映射可 保持不变。不过， 一旦主题中增加了分区，那么就 难以保证 key 与分区之间的映射关系了。