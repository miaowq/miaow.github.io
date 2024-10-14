---
title: Kafka指令
categories:
  - mq
tags:
  - mq
  - kafka
abbrlink: 931600eb
date: 2023-12-07 23:00:02
---

> 本系列内容基于 kafka_2.12-2.3.1

本节内容记录了Kafka相关指令

# 基础命令

**请一定注意Kafka版本，下列所有命令的版本最低为2.3.1**

<!-- more -->

## topic

```Shell
# 创建topic
kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic mytest

# 列出所有topic
kafka-topics.sh --list --bootstrap-server localhost:9092 

# 查看指定topic
kafka-topics.sh --describe --bootstrap-server localhost:9092 --topic mytest

# 删除Topic
kafka-topics.sh --bootstrap-server localhost:9092 --topic mytest --delete 

# 增加topic的partition数
kafka-topics.sh --bootstrap-server localhost:9092 --alter --topic mytest --partitions 5 

# 查看 topic 指定分区 offset 的最大值或最小值
# time 为 -1 时表示最大值，为 -2 时表示最小值
kafka-run-class.sh kafka.tools.GetOffsetShell --topic mytest --time -1 --broker-list localhost:9092 --partitions 0 
```

## 消息

```Shell
# 生产消息
kafka-console-producer.sh --broker-list localhost:9092 --topic mytest 

# 消费消息
# --from-beginning                                               从头开始消费
# --offset latest --partition {partition值} --max-messages {NUM} 从尾部开始取NUM个数据，且必需要指定分区

# 普通消费
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic mytest

# 从尾部开始，必需指定分区
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic mytest --offset latest --partition 0

# 取指定个数
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic mytest --offset latest --partition 0 --max-messages 1 
```