---
title: 安装Kafka和相关配置
categories:
  - mq
tags:
  - mq
  - kafka
abbrlink: ca0df96a
date: 2023-12-07 23:00:01
---

> 本系列内容基于 kafka_2.12-2.3.1
>
> 本系列示例语言：Golang
> 
> 本节内容以Linux环境搭建

# 安装与配置

本系列内容时基于 kafka_2.11-2.22，当前版本Kafka依赖Zookeeper，他们运行在 JVM 之上的服务，所以还需要安装 JDK。

> Kafka3 完全移除了zookeeper的依赖
> 
> Kafka 从2.0.0版本开始就不再支持 JDK7 及以下版本，本节就以 JDK8 为例来进行演示。

<!-- more -->

## JDK的安装

```Shell
wget https://repo.huaweicloud.com/java/jdk/8u181-b13/jdk-8u181-linux-x64.tar.gz

tar -zxvf jdk-8u181-linux-x64.tar.gz

mv jdk1.8.0_181/ /usr/local/java

# 设置环境变量
vim /etc/profile

# 添加下面内容
export JAVA_HOME=/usr/local/java
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JAVA_HOME/jre/lib/rt.jar
export PATH=$PATH:$JAVA_HOME/bin

# 使环境变量生效
source /etc/profile

# 验证是否安装完成
java -version
javac -version
```

## Zookeeper的安装与配置

Kafka 通过 ZooKeeper 来实施对元数据信息的管理，包括集群、broker、主题、分区等内容。

ZooKeeper 是一个开源的分布式协调服务，是 Google Chubby的一个开源实现。分布式应用程序可以基于 ZooKeeper 实现诸如数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Master 选举、配置维护等功能。

在 ZooKeeper 中共有3个角色：leader、follower 和 observer，同一时刻 ZooKeeper 集群中只会有一个 leader，其他的都是 follower 和 observer。observer 不参与投票，默认情况下 ZooKeeper 中只有 leader 和 follower 两个角色。更多相关知识可以查阅 ZooKeeper 官方网站来获得。

```Shell
wget https://mirrors.aliyun.com/apache/zookeeper/zookeeper-3.7.2/apache-zookeeper-3.7.2-bin.tar.gz

tar -zxvf apache-zookeeper-3.7.2-bin.tar.gz

mv apache-zookeeper-3.7.2-bin/ /usr/local/zookeeper

# 设置环境变量
vim /etc/profile

# 添加下面内容
export ZOOKEEPER_HOME=/usr/local/zookeeper
export PATH=$PATH:$ZOOKEEPER_HOME/bin

# 使环境变量生效
source /etc/profile

cd /usr/local/zookeeper/conf

cp zoo_sample.cfg zoo.cfg

vim zoo.cfg
```
```Shell
# ZooKeeper服务器心跳时间，单位为ms
tickTime=2000
# 允许follower连接并同步到leader的初始化连接时间，以tickTime的倍数来表示
initLimit=10
# leader与follower心跳检测最大容忍时间，响应超过syncLimit*tickTime，leader认为
# follower“死掉”，从服务器列表中删除follower
syncLimit=5
# 数据目录
dataDir=/usr/local/zookeeper/data
# 日志目录
dataLogDir=/usr/local/zookeeper/log
# ZooKeeper对外服务端口
clientPort=2181
```

```Shell
mkdir /usr/local/zookeeper/data
mkdir /usr/local/zookeeper/log

# 如果是集群模式
在${dataDir}目录（也就是/usr/local/zookeeper/data）下创建一个 myid 文件，并写入一个数值，比如0。myid 文件里存放的是服务器的编号。

cd /usr/local/zookeeper/bin/

./zkServer.sh start
./zkServer.sh status
```

以上是关于 ZooKeeper 单机模式的安装与配置，一般在生产环境中使用的都是集群模式，集群模式的配置也比较简单，相比单机模式而言只需要修改一些配置即可。下面以3台机器为例来配置一个 ZooKeeper 集群。首先在这3台机器的 /etc/hosts 文件中添加3台集群的IP地址与机器域名的映射，示例如下（3个IP地址分别对应3台机器）：

```Shell
192.168.0.2 node1
192.168.0.3 node2
192.168.0.4 node3
```

然后在这3台机器的 zoo.cfg 文件中添加以下配置：

```Shell
server.0=192.168.0.2:2888:3888
server.1=192.168.0.3:2888:3888
server.2=192.168.0.4:2888:3888
```

为了便于讲解上面的配置，这里抽象出一个公式，即 server.A=B:C:D。其中A是一个数字，代表服务器的编号，就是前面所说的 myid 文件里面的值。集群中每台服务器的编号都必须唯一，所以要保证每台服务器中的 myid 文件中的值不同。B代表服务器的IP地址。C表示服务器与集群中的 leader 服务器交换信息的端口。D表示选举时服务器相互通信的端口。如此，集群模式的配置就告一段落，可以在这3台机器上各自执行 zkServer.sh start 命令来启动服务。

## Kafka的安装与配置

```Shell
wget https://archive.apache.org/dist/kafka/2.2.2/kafka_2.11-2.2.2.tgz

tar -zxvf kafka_2.11-2.2.2.tgz

mv kafka_2.11-2.2.2/ /usr/local/kafka

# 设置环境变量
vim /etc/profile

# 添加下面内容
export KAFKA_HOME=/usr/local/kafka
export PATH=$PATH:$KAFKA_HOME/bin

# 使环境变量生效
source /etc/profile

cd /usr/local/kafka/conf

vim server.properties

# 主要关注以下几个配置参数即可：
```

```Shell
# broker的编号，如果集群中有多个broker，则每个broker的编号需要设置的不同
broker.id=0
# broker对外提供的服务入口地址
listeners=PLAINTEXT://localhost:9092
# 存放消息日志文件的地址
log.dirs=/usr/local/kafka/log
# Kafka所需的ZooKeeper集群地址，为了方便演示，我们假设Kafka和ZooKeeper都安装在本机
zookeeper.connect=localhost:2181/kafka

mkdir /usr/local/kafka/log
```

如果是单机模式，那么修改完上述配置参数之后就可以启动服务。如果是集群模式，那么只需要对单机模式的配置文件做相应的修改即可：确保集群中每个 broker 的 broker.id 配置参数的值不一样，以及 listeners 配置参数也需要修改为与 broker 对应的IP地址或域名，之后就可以各自启动服务。注意，在启动 Kafka 服务之前同样需要确保 zookeeper.connect 参数所配置的 ZooKeeper 服务已经正确启动。

启动 Kafka 服务的方式比较简单，在$KAFKA_HOME 目录下执行下面的命令即可：

```Shell
cd /usr/local/kafka/bin

# 前台启动
./ kafka-server-start.sh /usr/local/kafka/conf/server.properties

# 后台启动
./ kafka-server-start.sh -daemon /usr/local/kafka/conf/server.properties
./ kafka-server-start.sh  /usr/local/kafka/conf/server.properties &
```

可以通过 jps 命令查看 Kafka 服务进程是否已经启动，示例如下：

```Shell
jps -l

23152 sun.tools.jps.Jps
16052 org.apache.zookeeper.server.quorum.QuorumPeerMain
22807 kafka.Kafka  # 这个就是Kafka服务端的进程
```

> 注意：jps 命令只是用来确认 Kafka 服务的进程已经正常启动。它是否能够正确地对外提供服务，还需要通过发送和消费消息来进行验证，验证的过程可以参考下面的内容。

# 生产和消费

根据上一节内容可知，生产者将消息发送至 Kafka 的主题中，或者更加确切地说应该是主题的分区中，而消费者也是通过订阅主题从而消费消息的。在演示生产与消费消息之前，需要创建一个主题作为消息的载体。

## 创建topic

Kafka 提供了许多实用的脚本工具，存放在 $KAFKA_HOME 的 bin 目录下，其中与主题有关的就是 kafka-topics.sh 脚本，下面我们用它演示创建一个分区数为4、副本因子为3的主题 topic-demo，示例如下（Kafka集群模式下，broker数为3）：

```Shell
# --bootstrap-server   指定了 Kafka 所连接的 ZooKeeper 服务地址
# --topic              指定了所要创建主题的名称
# --replication-factor 指定了副本因子
# --partitions         指定了分区个数
# --create             是创建主题的动作指令
/usr/local/kafka/bin/kafka-topics.sh --bootstrap-server localhost:2181/kafka --create --topic topic-demo --replication-factor 3 --partitions 4
```

还可以通过 --describe 展示主题的更多具体信息，示例如下：

```Shell
./kafka-topics.sh --bootstrap-server localhost:2181/kafka --describe --topic topic-demo

Topic:topic-demo	PartitionCount:4	ReplicationFactor:3	Configs:
	Topic: topic-demo	Partition: 0	Leader: 2	Replicas: 2,1,0	Isr: 2,0
	Topic: topic-demo	Partition: 1	Leader: 2	Replicas: 0,2,1	Isr: 2
	Topic: topic-demo	Partition: 2	Leader: 2	Replicas: 1,0,2	Isr: 2
	Topic: topic-demo	Partition: 3	Leader: 2	Replicas: 2,0,1	Isr: 2,0
```

## 发送和消费消息

kafka-console-producer.sh 和 kafka-console-consumer.sh，通过控制台收发消息

### 打开消费端

```Shell
# --bootstrap-server 指定了连接的 Kafka 集群地址
# --topic            指定了消费者订阅的主题
./kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic topic-demo
```

### 打开生产端
```Shell
# --broker-list 指定了连接的 Kafka 集群地址
# --topic 指定了发送消息时的主题
./kafka-console-producer.sh --broker-list localhost:9092 --topic topic-demo
>Hello, Kafka!
```

示例中的第二行是通过人工键入的方式输入的，按下回车键后会跳到第三行，即“>”字符处。此时原先执行 kafka-console-consumer.sh 脚本的 shell 终端中出现了刚刚输入的消息“Hello, Kafka!”，示例如下：

```Shell
./kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic topic-demo
Hello, Kafka!
```

以下是简易代码实例

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

	if err := writer.WriteMessages(
		ctx,
		kafka.Message{
			Key:   []byte("1"),
			Value: []byte("Hello, kafka"),
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
			fmt.Printf(
				"topic=%s, partition=%d, offset=%d, key=%s, value=%s\n",
				msg.Topic, msg.Partition, msg.Offset, msg.Key, msg.Value,
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

# 服务端参数配置

前面的 Kafka 安装与配置的说明中只是简单地表述了几个必要的服务端参数而没有对其进行详细的介绍，并且 Kafka 服务端参数（broker configs）也并非只有这几个。Kafka 服务端还有很多参数配置，涉及使用、调优的各个方面，虽然这些参数在大多数情况下不需要更改，但了解这些参数，以及在特殊应用需求的情况下进行有针对性的调优，可以更好地利用 Kafka 为我们工作。

下面挑选一些重要的服务端参数来做细致的说明，这些参数都配置在$KAFKA_HOME/config/server.properties 文件中。

## zookeeper.connect
该参数指明 broker 要连接的 ZooKeeper 集群的服务地址（包含端口号），没有默认值，且此参数为必填项。可以配置为 localhost:2181，如果 ZooKeeper 集群中有多个节点，则可以用逗号将每个节点隔开，类似于 localhost1:2181,localhost2:2181,localhost3:2181 这种格式。最佳的实践方式是再加一个 chroot 路径，这样既可以明确指明该 chroot 路径下的节点是为 Kafka 所用的，也可以实现多个 Kafka 集群复用一套 ZooKeeper 集群，这样可以节省更多的硬件资源。包含 chroot 路径的配置类似于 localhost1:2181,localhost2:2181,localhost3:2181/kafka 这种，如果不指定 chroot，那么默认使用 ZooKeeper 的根路径。

## listeners
该参数指明 broker 监听客户端连接的地址列表，即为客户端要连接 broker 的入口地址列表，配置格式为 protocol1://hostname1:port1,protocol2://hostname2:port2，其中 protocol 代表协议类型，Kafka 当前支持的协议类型有 PLAINTEXT、SSL、SASL_SSL 等，如果未开启安全认证，则使用简单的 PLAINTEXT 即可。hostname 代表主机名，port 代表服务端口，此参数的默认值为 null。比如此参数配置为 PLAINTEXT://198.162.0.2:9092，如果有多个地址，则中间以逗号隔开。如果不指定主机名，则表示绑定默认网卡，注意有可能会绑定到127.0.0.1，这样无法对外提供服务，所以主机名最好不要为空；如果主机名是0.0.0.0，则表示绑定所有的网卡。

与此参数关联的还有 advertised.listeners，作用和 listeners 类似，默认值也为 null。不过 advertised.listeners 主要用于 IaaS（Infrastructure as a Service）环境，比如公有云上的机器通常配备有多块网卡，即包含私网网卡和公网网卡，对于这种情况而言，可以设置 advertised.listeners 参数绑定公网IP供外部客户端使用，而配置 listeners 参数来绑定私网IP地址供 broker 间通信使用。

## broker.id
该参数用来指定 Kafka 集群中 broker 的唯一标识，默认值为-1。如果没有设置，那么 Kafka 会自动生成一个。这个参数还和 meta.properties 文件及服务端参数 broker.id.generation. enable 和 reserved.broker.max.id 有关，相关深度解析可以参考《图解Kafka之核心原理》的相关内容。

## log.dir和log.dirs
Kafka 把所有的消息都保存在磁盘上，而这两个参数用来配置 Kafka 日志文件存放的根目录。一般情况下，log.dir 用来配置单个根目录，而 log.dirs 用来配置多个根目录（以逗号分隔），但是 Kafka 并没有对此做强制性限制，也就是说，log.dir 和 log.dirs 都可以用来配置单个或多个根目录。log.dirs 的优先级比 log.dir 高，但是如果没有配置 log.dirs，则会以 log.dir 配置为准。默认情况下只配置了 log.dir 参数，其默认值为 /tmp/kafka-logs。

## message.max.bytes
该参数用来指定 broker 所能接收消息的最大值，默认值为1000012（B），约等于976.6KB。如果 Producer 发送的消息大于这个参数所设置的值，那么（Producer）就会报出 RecordTooLargeException 的异常。如果需要修改这个参数，那么还要考虑 max.request.size（客户端参数）、max.message.bytes（topic端参数）等参数的影响。为了避免修改此参数而引起级联的影响，建议在修改此参数之前考虑分拆消息的可行性。

还有一些服务端参数在本节没有提及，这些参数同样非常重要，它们需要用单独的章节或者场景来描述，比如 unclean.leader.election.enable、log.segment.bytes 等参数都会在后面的章节中提及。
