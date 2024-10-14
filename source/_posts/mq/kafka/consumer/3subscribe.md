---
title: 消费者｜客户端开发
categories:
  - mq
tags:
  - mq
  - kafka
abbrlink: d51cf919
date: 2023-12-08 23:00:02
---

> 本系列内容基于 kafka_2.12-2.3.1
> 
> Golang Java

# 必要的参数配置

**bootstrap.servers**

该参数的释义和生产者客户端 KafkaProducer 中的相同，用来指定连接 Kafka 集群所需的 broker 地址清单，具体内容形式为 host1:port1,host2:post，可以设置一个或多个地址，中间用逗号隔开，此参数的默认值为""

<!-- more -->

注意这里并非需要设置集群中全部的 broker 地址，消费者会从现有的配置中查找到全部的 Kafka 集群成员。这里设置两个以上的 broker 地址信息，当其中任意一个宕机时，消费者仍然可以连接到 Kafka 集群上。

**group.id**

消费者隶属的消费组的名称，默认值为""

一般而言，这个参数需要设置成具有一定的业务意义的名称。

**非必需 client.id**

设定 KafkaConsumer 对应的客户端id，默认值为""

如果客户端不设置，则 KafkaConsumer 会自动生成一个非空字符串，内容形式如“consumer-1”、“consumer-2”，即字符串“consumer-”与数字的拼接。

**Java中必需 key.deserializer 和 value.deserializer**

与生产者客户端 KafkaProducer 中的 key.serializer和value.serializer 参数对应。消费者从 broker 端获取的消息格式都是字节数组（byte[]）类型，所以需要执行相应的反序列化操作才能还原成原有的对象格式。这两个参数分别用来指定消息中 key 和 value 所需反序列化操作的反序列化器，**这两个参数无默认值**。

注意这里必须填写反序列化器类的全限定名，比如示例中的 org.apache.kafka.common.serialization.StringDeserializer，单单指定 StringDeserializer 是错误的。

# 订阅主题和分区

一个消费者可以订阅一个或多个主题。

在Java中，使用如下方式订阅主题：

```Java
consumer.subscribe(Arrays.asList(topic1));
consumer.subscribe(Arrays.asList(topic2));
```

则：消费者订阅的是 topic2，而不是 topic1，也不是 topic1 和 topic2 的并集。

Java中 `subscribe`有4个重载方法，自行查阅。

```Java
public void subscribe(Collection<String> topics, 
    ConsumerRebalanceListener listener)
public void subscribe(Collection<String> topics)
public void subscribe(Pattern pattern, ConsumerRebalanceListener listener)
public void subscribe(Pattern pattern)
```

Java中通过如下方式订阅某个主题的某个分区：

```Java
consumer.assign(Arrays.asList(new TopicPartition("topic-demo", 0)));

// 如果不知道有多少个分区怎么办？自行查阅
```

在Golang中相对简单：

```Golang
config := kafka.ReaderConfig{
  Brokers:        []string{"xxx"},
  Topic:          topic, // string 主题，如 "test0,test1"
  GroupID:        "xxx",
  Partition:      1, // int 分区
}
reader = kafka.NewReader(config)
```

# 几个问题

**1. assign 和 subscribe 的区别**

他们是两种不同的方式来指定消费者要处理的分区。主要区别在于分区的分配方式和对消费者组协调的依赖。

**assign**
- 消费者手动指定要处理的分区。
- 消费者绕过了协调器的分区分配机制，直接指定了要处理的分区，不再依赖于消费者组的协调。
- 消费者需要提供一个 `TopicPartition` 列表，其中包含要分配给该消费者的分区。
- 消费者将不再参与消费者组协调器的协调，因此不会触发rebalance操作。
- 手动分配分区可能导致负载不均衡，因此需要确保手动指定的分区数与消费者数量匹配，并且需要自行处理分区的重新分配。

**subscribe**
- Kafka协调器会自动进行分配。
- 消费者加入到一个或多个主题的一个消费者组中。
- Kafka协调器负责为每个消费者分配分区，并确保每个分区只分配给消费者组内的一个消费者。