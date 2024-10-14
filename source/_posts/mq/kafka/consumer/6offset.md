---
title: 消费者｜位移or偏移量
categories:
  - mq
tags:
  - mq
  - kafka
abbrlink: 75abb0fe
date: 2023-12-08 23:00:05
---

> 本系列内容基于 kafka_2.12-2.3.1

# 2个概念

对于**分区**而言，它的每条消息都有唯一的 `offset`，用来表示**消息在分区中对应的位置**。

对于**消费者**而言，它也有一个 `offset` 的概念，消费者使用 `offset` 来表示**消费到分区中某个消息所在的位置**。

因此，在下文中：

对于消息在分区中的位置，我们将 `offset` 称为 `偏移量`

对于消费者消费到的位置，将 `offset` 称为 `位移` 或 `消费位移`

<!-- more -->

# 存储位置

在每次调用 `poll()`方法时，它返回的是还没有被消费过的消息集。要做到这一点，就需要记录上一次消费时的`消费位移`。

再考虑一种情况，当有新的消费者加入时，那么必然会有`再均衡`的动作，对于同一分区而言，它可能在`再均衡`动作之后分配给`新的消费者`，如果不持久化保存消费位移，那么这个新的消费者也无法知晓之前的消费位移。

因此这个`消费位移`必须做`持久化保存`，而不是单单保存在内存中，否则消费者重启之后就无法知晓之前的消费位移。

在旧消费者客户端中（< 0.9版本），消费位移是存储在 `ZooKeeper` 中的。而在新消费者客户端中，消费位移存储在 Kafka 内部的`主题__consumer_offsets` 中。（不让你消费者和Zookeeper网络通信太过频繁）

这里把将`消费位移`存储起来（持久化）的动作称为`提交`，消费者在消费完消息之后需要执行`消费位移`的`提交`。

![](/images/mq/kafka/kafka12.png)

参考上图中的消费位移，`x` 表示某一次拉取操作中此分区消息的最大偏移量，假设当前消费者已经消费了 `x` 位置的消息，那么我们就可以说消费者的`消费位移`为 `x`，图中也用了 `lastConsumedOffset` 这个单词来标识它。

# 提交位置

以上图为例，消费者**提交的消费位移并不是x，而是x+1**，也就是对应于上图中的 position，它表示**下一条需要拉取的消息的位置**。在消费者中还有一个 committed offset 的概念，它表示已经提交过的消费位移。

> 类似的还有LEO，可以参考 初识卡夫卡

**注意：position 和 committed offset 并不会一直相同**，如：

![](/images/mq/kafka/kafka13.png)

## 重复消费和消息丢失
对于位移提交的具体时机的把握也很有讲究，有可能会造成重复消费和消息丢失的现象。

从图中可以明确的是：
- <= x + 1 的消息已经消费完了
- 正在处理 x + 5 的消息
- 正在提交 x + 8 的消息

那么：

如果拉取到消息就进行位移提交，即提交了 `x + 8`，那么假设 `x + 5`**消费异常**，故障恢复后，重新拉取消息会从 `x + 8` 开始。那么`[x + 5:x + 7]`的消息并未能被消费，如此便发生了`消息丢失`的现象。

如果在消费完所有拉取到的消息之后才执行位移提交，那么假设 `x + 5`**消费异常**，故障恢复后，重新拉取消息会从 `x + 2` 开始。也就是说`[x + 2:x + 4]`的消息又重新消费了一遍，故而又发生了`重复消费`的现象。

而实际情况可能比这个要复杂得多...

# 提交方式

## 自动提交

### 影响参数

`enable.auto.commit` 默认为true，当然这个默认的自动提交不是每消费一条消息就提交一次，而是定期提交，受下面参数控制
`auto.commit.interval.ms` 默认值为5秒，此参数生效的前提是 enable.auto.commit 参数为 true 。

在默认的方式下，消费者每隔5秒会将拉取到的每个分区中最大的消息位移进行提交。自动位移提交的动作是在 poll() 方法的逻辑里完成的，在每次真正向服务端发起拉取请求之前会检查是否可以进行位移提交，如果可以，那么就会提交上一次轮询的位移。

### 重复消费和消息丢失

在 Kafka 消费的编程逻辑中位移提交是一大难点，自动提交消费位移的方式非常简便，它免去了复杂的位移提交逻辑，让编码更简洁。但随之而来的是重复消费和消息丢失的问题。

`重复消费`：假设刚刚提交完一次消费位移，然后拉取一批消息进行消费，在下一次自动提交消费位移之前，消费者崩溃了，那么又得从上一次位移提交的地方重新开始消费，这样便发生了重复消费的现象（对于再均衡的情况同样适用）。我们可以通过减小位移提交的时间间隔来减小重复消息的窗口大小，但这样并不能避免重复消费的发送，而且也会使位移提交更加频繁。

![](/images/mq/kafka/kafka14.png)

`消息丢失`上图描述：
- 线程A 不断地拉取消息并存入本地缓存（BlockingQueue）
- 线程B 从缓存中读取消息并进行相应的逻辑处理
- 当前进行到了 y + 1 次拉取，且第m次位移提交的时候，说明 <= x + 6的消息已经消费完成了

但此时 线程B 却还在消费 `x + 3` 的消息，如果 线程B 发生了异常，待其恢复后，会从第 m 次位移提交处，也就是 `x+6` 的位置开始拉取消息。也就是说`[x + 3:x + 6]`的消息没有得到相应处理，如此便发生了`消息丢失`的现象。

## 手动提交

自动位移提交的方式在正常情况下不会发生消息丢失或重复消费的现象，但是在编程的世界里异常无可避免，与此同时，自动位移提交也无法做到精确的位移管理。在 Kafka 中还提供了手动位移提交的方式，这样可以使得开发人员对消费位移的管理控制更加灵活。

很多时候并不是说拉取到消息就算消费完成，而是需要将消息写入数据库、写入本地缓存，或者是更加复杂的业务处理。

在这些场景下，所有的业务处理完成才能认为消息被成功消费，手动的提交方式可以让开发人员根据程序的逻辑在合适的地方进行位移提交。

### 影响参数

`enable.auto.commit` 配置为 false

### 同步提交和异步提交

```Golang

func read(ctx context.Context) {
	reader = kafka.NewReader(kafka.ReaderConfig{
		Brokers:        []string{"localhost:9092"},
		Topic:          topic,
		CommitInterval: 1 * time.Second,
		GroupID:        "order",
		StartOffset:    kafka.FirstOffset,
	})
	// 异常退出，defer执行不到
	// defer reader.Close()

	for {
    var offsetsToCommit []kafka.Message
		msg, err := reader.ReadMessage(ctx)

		if err != nil {
			fmt.Printf("读kafka失败%v\n", err)
			break
		}

		fmt.Printf(
			"topic=%s, partition=%d, offset=%d, key=%s, value=%s\n",
			msg.Topic, msg.Partition, msg.Offset, msg.Key, msg.Value,
		)

		// 同步提交
    reader.CommitMessages(ctx, msg)

    // 异步提交
    go func(msg kafka.Message) {
      reader.CommitMessages(ctx, msg)
    }(msg)

    // 批量提交
    // 将消息的偏移量添加到待提交列表
    offsetsToCommit = append(offsetsToCommit, msg)

    // 在这里判断何时执行批量提交，比如达到一定数量或者一定时间间隔
    if len(offsetsToCommit) >= 10 {
      reader.CommitMessages(ctx, offsetsToCommit...)
      offsetsToCommit = nil
    }
	}
}

func listenSignal() {
	ch := make(chan os.Signal, 1)

	signal.Notify(ch, syscall.SIGINT, syscall.SIGTERM)
	sig := <-ch

	fmt.Printf("接收到信号%s", sig.String())
	if reader != nil {
		reader.Close()
	}

	os.Exit(0)
}
```

## 控制或关闭消费
在有些应用场景下我们可能需要暂停某些分区的消费而先消费其他分区，当达到一定条件时再恢复这些分区的消费。

在 Java 中使用 pause() 和 resume() 方法来分别实现暂停某些分区在拉取操作时返回数据给客户端和恢复某些分区向客户端返回数据的操作。在Golang中，可以通过设置 ReaderConfig 中的 MaxBytes 字段来模拟暂停和恢复的操作：

- 当 MaxBytes 设为 0 时，消费者将在尝试读取数据时一直被阻塞，从而实现暂停的效果。
- 当 MaxBytes 设置为一个较大的值，可以恢复正常的数据拉取操作。

### 举个例子

定时任务每隔 5 分钟更新一次消费者状态：
```Golang
package main

import (
	"context"
	"fmt"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/segmentio/kafka-go"
)

var (
	reader   *kafka.Reader
	topic    = "test"
	minBytes = int(10e3)
	maxBytes = int(10e6)
)

type ConsumerState struct {
	Paused   bool
	MaxBytes int
}

func write(ctx context.Context) {
	writer := kafka.Writer{
		Addr:                   kafka.TCP("localhost:9092"),
		Topic:                  topic,
		Balancer:               &kafka.Hash{},
		WriteTimeout:           1 * time.Second,
		RequiredAcks:           kafka.RequireOne,
		AllowAutoTopicCreation: false,
	}
	defer writer.Close()

	key := "1"
	value := "Hello, kafka"
	if err := writer.WriteMessages(
		ctx,
		kafka.Message{
			Key:   []byte(key),
			Value: []byte(value),
		},
	); err != nil {
		if err == kafka.LeaderNotAvailable {
			time.Sleep(500 * time.Millisecond)
			return
		} else {
			fmt.Printf("写入kafka失败%v\n", err)
		}
	}

	fmt.Printf("写入kafka成功%s\n", value)
}

func read(ctx context.Context) {
	cs := initConsumerState()
	readerCfg := kafka.ReaderConfig{
		Brokers:        []string{"localhost:9092"},
		Topic:          topic,
		CommitInterval: 1 * time.Second,
		GroupID:        "order",
		StartOffset:    kafka.FirstOffset,
		MinBytes:       minBytes,
		MaxBytes:       maxBytes,
	}

	reader = kafka.NewReader(readerCfg)
	// 异常退出，defer执行不到
	// defer reader.Close()

	go func() {
		ticker := time.NewTicker(5 * time.Minute)
		defer ticker.Stop()

		for {
			select {
			case <-ticker.C:
				updateConsumerState(cs)
				readerCfg.MaxBytes = cs.MaxBytes
				fmt.Printf("Consumer state updated. Paused: %v, MaxBytes: %d\n", cs.Paused, cs.MaxBytes)
			}
		}
	}()

	for {
		if cs.Paused {
			time.Sleep(time.Second)
		} else {
			msg, err := reader.ReadMessage(ctx)

			if err != nil {
				fmt.Printf("读kafka失败%v\n", err)
				break
			}

			fmt.Printf(
				"topic=%s, partition=%d, offset=%d, key=%s, value=%s\n",
				msg.Topic, msg.Partition, msg.Offset, msg.Key, msg.Value,
			)

			reader.CommitMessages(ctx, msg)
		}
	}

}

func listenSignal() {
	ch := make(chan os.Signal, 1)

	signal.Notify(ch, syscall.SIGINT, syscall.SIGTERM)
	sig := <-ch

	fmt.Printf("接收到信号%s", sig.String())
	if reader != nil {
		reader.Close()
	}
	os.Exit(0)
}

func initConsumerState() *ConsumerState {
	return &ConsumerState{
		Paused:   false,
		MaxBytes: maxBytes,
	}
}

func updateConsumerState(cs *ConsumerState) {
	cs.Paused = !cs.Paused
	if cs.Paused {
		cs.MaxBytes = 0 // 暂停状态，将 MaxBytes 设置为 0
	} else {
		cs.MaxBytes = maxBytes // 恢复状态，将 MaxBytes 设置为一个较大的值
	}
}

func main() {
	ctx := context.Background()

	write(ctx)
	go listenSignal()
	read(ctx)
}
```

### 关闭消费｜优雅退出

```Golang
func read(ctx context.Context) {
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()

	reader = kafka.NewReader(kafka.ReaderConfig{
		Brokers:        []string{"localhost:9092"},
		Topic:          topic,
		CommitInterval: 1 * time.Second,
		GroupID:        "order",
		StartOffset:    kafka.FirstOffset,
	})
	// 异常退出，defer执行不到
	// defer reader.Close()

	select {
	case <-ctx.Done():
		fmt.Println("Context canceled to do something")
		return
	default:
		if msg, err := reader.ReadMessage(ctx); err != nil {
			fmt.Printf("读kafka失败%v\n", err)
		} else {
			fmt.Printf(
				"topic=%s, partition=%d, offset=%d, key=%s, value=%s\n",
				msg.Topic, msg.Partition, msg.Offset, msg.Key, msg.Value,
			)
		}
	}
}
```

# 指定位移消费

## 找不到消费位移

当一个新的消费组建立的时候，它根本没有可以查找的消费位移。

或者消费组内的一个新消费者订阅了一个新的主题，它也没有可以查找的消费位移。

当 __consumer_offsets `主题`中有关这个消费组的位移信息过期而被删除后，它也没有可以查找的消费位移。

## auto.offset.reset

在 Kafka 中每当消费者查找不到所记录的消费位移时，就会根据消费者客户端参数 `auto.offset.reset` 的配置来决定从何处开始进行消费，这个参数的默认值为 `latest` ，表示从分区末尾开始消费消息。

![](/images/mq/kafka/kafka15.png)

参考上图，按照默认的配置，消费者会从9开始进行消费（9是下一条要写入消息的位置），更加确切地说是从9开始拉取消息。

如果将 auto.offset.reset 参数配置为 `earliest`，那么消费者会从起始处，也就是0开始消费。

除了查找不到消费位移，位移越界也会触发 auto.offset.reset 参数的执行，这个在下面要讲述的 seek 系列的方法中会有相关的介绍。

auto.offset.reset 参数还有一个可配置的值—“none”，配置为此值就意味着出现查到不到消费位移的时候，既不从最新的消息位置处开始消费，也不从最早的消息位置处开始消费，此时会报出 NoOffsetForPartitionException 异常，示例如下：

```Java
org.apache.kafka.clients.consumer.NoOffsetForPartitionException: Undefined offset with no reset policy for partitions: [topic-demo-3, topic-demo-0, topic-demo-2, topic-demo-1].
```

如果能够找到消费位移，那么配置为“none”不会出现任何异常。如果配置的不是“latest”、“earliest”和“none”，则会报出 ConfigException 异常，示例如下：

```Java
org.apache.kafka.common.config.ConfigException: Invalid value any for configuration auto.offset.reset: String must be one of: latest, earliest, none.
```

到目前为止，我们知道消息的拉取是根据 poll() 方法中的逻辑来处理的，这个 poll() 方法中的逻辑对于普通的开发人员而言是一个黑盒，无法精确地掌控其消费的起始位置。提供的 auto.offset.reset 参数也只能在找不到消费位移或位移越界的情况下粗粒度地从开头或末尾开始消费。有些时候，我们需要一种更细粒度的掌控，可以让我们从特定的位移处开始拉取消息，而 KafkaConsumer 中的 seek() 方法正好提供了这个功能，让我们得以追前消费或回溯消费。

```Golang
// 指定offset
reader = kafka.NewReader(kafka.ReaderConfig{
	Brokers:        []string{"localhost:9092"},
	Topic:          topic,
	CommitInterval: 1 * time.Second,
	// 如果指定了，会返回错误 https://github.com/segmentio/kafka-go?tab=readme-ov-file#consumer-groups
	//GroupID:   "xxxx",
	Partition: ctx.Value("partition").(int),
})

err := reader.SetOffset(10)
if err != nil {
	fmt.Printf("Set offset error: %v\n", err)
	return
}

// 指定时间
reader.SetOffsetAt(ctx, time.Now().Add(-time.Hour))
if msg.Time.After(time.Now()) {
	break
}
```