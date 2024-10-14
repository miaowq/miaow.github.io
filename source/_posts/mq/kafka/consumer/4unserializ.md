---
title: 消费者｜反序列化
categories:
  - mq
tags:
  - mq
  - kafka
abbrlink: 6c74740b
date: 2023-12-08 23:00:03
---

> 本系列内容基于 kafka_2.12-2.3.1

生产者需要用序列化器（Serializer）把对象转换成字节数组才能通过网络发送给 Kafka。而在对侧，消费者需要用反序列化器（Deserializer）把从 Kafka 中收到的字节数组转换成相应的对象。

<!-- more -->

之前的示例代码中，为了方便，消息的 key 和 value 都使用了字符串，如果是对象就需要用到 `github.com/golang/protobuf/proto`包，具体使用可自行查阅。

```Golang
package main

import (
	"context"
	"fmt"
	"github.com/golang/protobuf/proto"
	"os"
	"os/signal"
	"syscall"
	"time"

	myProto "myProto所在路径"

	"github.com/segmentio/kafka-go"
)

var (
	reader *kafka.Reader
	topic  = "test"
)

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

	value, _ := proto.Marshal(&myProto.User{
		Id:   1,
		Name: "hello, proto Kafka!",
	})
	key := "1"
	if err := writer.WriteMessages(
		ctx,
		kafka.Message{
			Key:   []byte(key),
			Value: value,
		},
	); err != nil {
		if err == kafka.LeaderNotAvailable {
			time.Sleep(500 * time.Millisecond)
			return
		} else {
			fmt.Printf("写入kafka失败%v\n", err)
		}
	}
}

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
		if msg, err := reader.ReadMessage(ctx); err != nil {
			fmt.Printf("读kafka失败%v\n", err)
		} else {
			u := &myProto.User{}
			err := proto.Unmarshal(msg.Value, u)
			if err != nil {
				return
			}

			fmt.Printf(
				"topic=%s, partition=%d, offset=%d, key=%s\n",
				msg.Topic, msg.Partition, msg.Offset, msg.Key,
			)
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

func main() {
	ctx := context.Background()

	write(ctx)
	go listenSignal()
	read(ctx)
}

```