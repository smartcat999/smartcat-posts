---
title: "Kafka数据存储"
draft: false
toc: true
description: "KafkaOperator基于connector/user/topic的管理配置"
date: 2022-10-23
featured_image:
categories: []
tags: [kafka, storage]
---
Kafka 数据存储分析与测试<!--more-->
##### 1 kafka 数据存储格式

![图片](images/msg.png)
**字段含义**

| 字段名             | 字段含义                                                         |
|-----------------|--------------------------------------------------------------|
| baseOffset      | 消息的起始offset                                                  |
| batchLength     | 消息数量                                                         |
| magic           | 用于扩展信息，当前版本为2                                                |
| crc             | crc校验                                                        |
| attributes      | int16, 0~2位表示压缩算法 3位表示时间戳类型，4位表示是否使用事务，5位表示是否是控制消息， 6～15暂时没用 |
| lastOffsetDelta | BatchRecords中最后一条消息相对与baseOffset的值                           |
| firstTimestamp  | BatchRecords中最早一条消息的时间戳                                      |
| maxTimestamp    | BatchRecords中最新一条消息的时间戳                                      |
| producerId      | 生产者ID                                                        |
| producerEpoch   | 支持生产者消息幂等                                                    |
| baseSequence    | 起始消息序列号                                                      |
| records         | 消息内容                                                         |

##### 2 存储空间大小计算

**消息大小 * 当天消息数 * 消息保留天数 * 消息副本数 * （ 1 + 索引文件百分比 ） * 压缩比 * （1 + 预留空间百分比） / 1024 /
1024 / 1024 TB
例如：1KB * 1亿 * 7天 * 3个副本 * (1 + 10%) * 0.8 / 1024 / 1024 / 1024 约等于 1.8TB， 一天1亿的消息量，每条消息大小1KB，topic3个副本，
索引文件比例按 10% ～
20%， 压缩比率按 10 % ～ 20%，压缩之后文件大小为源文件大小80% ～ 90%， 计算范围在 1.8TB ～ 2.2TB， 预留10% ～ 20%空间，安全阈值
2.0TB ～ 2.7TB。    
按1千个设备，每个设备20个测点，一共2w个测点，一秒钟上报一次数据计算， 单条消息大小 1KB， 每秒的推送消息大小 1KB，
kafka内部分发一次，每秒产生的消息大小 1KB * 2 = 2KB， 单个设备每天的推送消息大小 2 *
60 * 60 * 24 = 172800KB，约等于168.75MB， 所有设备每天累计推送消息大小 172800KB * 1000个 = 172800000KB，约等于164.80GB,
按公式计算 3045.41GB ～
3737.55GB,换算为 2.97TB ～ 3.65TB，加上预留空间10%，安全阈值 3.27TB ～ 4.02TB。**

##### 3 kafka消息写入存储空间测试

###### 3.1 安装kafka(helm)

```shell
# 安装kafka
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm install kafka-master bitnami/kafka -n ps

# 安装kafka-client
$ kubectl run kafka-master-client --restart='Never' --image docker.io/bitnami/kafka:3.2.0-debian-10-r4 --namespace ps --command -- sleep infinity

# 进入容器
$ kubectl exec --tty -i kafka-master-client --namespace ps -- bash

# 使用client调用kafka-server
# 设置环境变量kafka-server地址
$ export kafka=kafka-master-0.kafka-master-headless.ps.svc.cluster.local:9092

$ cd /opt/bitnami/kafka/bin
# 创建topic
$ kafka-topics.sh --create --topic test --bootstrap-server $kafka

# 启动 consumer-console
$ kafka-console-consumer.sh --topic test --from-beginning --bootstrap-server $kafka

# 启动 producer-console
$ kafka-console-producer.sh --topic test --bootstrap-server $kafka

# 查看日志文件
$ /opt/bitnami/kafka/bin/kafka-run-class.sh kafka.tools.DumpLogSegments --files 00000000000000000000.log --print-data-log
$ /opt/bitnami/kafka/bin/kafka-dump-log.sh --files 00000000000000000000.log --print-data-log
# output:
baseOffset: 45 lastOffset: 45 count: 1 baseSequence: 0 lastSequence: 0 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional: false isControl: false deleteHorizonMs: OptionalLong.empty position: 2031009 CreateTime: 1654672883918 size: 331 magic: 2 compresscodec: gzip crc: 4084402132 isvalid: true
| offset: 45 CreateTime: 1654672883918 keySize: -1 valueSize: 389 sequence: 0 headerKeys: [] payload: {"widget":{"debug":"on","window":{"title":"Sample Konfabulator Widget","name":"main_window","width":500,"height":500},"image":{"src":"Images/Sun.png","name":"sun1","hOffset":250,"vOffset":250,"alignment":"center"},"text":{"data":"Click Here","size":36,"style":"bold","name":"text1","hOffset":250,"vOffset":100,"alignment":"center","onMouseUp":"sun1.opacity = (sun1.opacity / 100) * 90;"}}}

# 查看索引文件
$ /opt/bitnami/kafka/bin/kafka-dump-log.sh --files 00000000000000000000.index
$ /opt/bitnami/kafka/bin/kafka-dump-log.sh --files 00000000000000000000.timeindex
```

###### 3.2 写入测试

```go
// consumer.go
package kafka_demo

import (
	"context"
	"github.com/Shopify/sarama"
	"log"
	"os"
	"os/signal"
	"strings"
	"sync"
	"syscall"
)

type KafkaClient struct {
	topics        []string
	consumerGroup sarama.ConsumerGroup
}

func NewKafkaClient(brokers string, topics string, groupID string, oldest bool, version string, assignor string, verbose bool) *KafkaClient {
	if verbose {
		sarama.Logger = log.New(os.Stdout, "[sarama] ", log.LstdFlags)
	}

	config := sarama.NewConfig()
	ver, err := sarama.ParseKafkaVersion(version)
	if err != nil {
		log.Panic(err)
	}
	config.Version = ver

	switch assignor {
	case "sticky":
		config.Consumer.Group.Rebalance.Strategy = sarama.BalanceStrategySticky
	case "roundrobin":
		config.Consumer.Group.Rebalance.Strategy = sarama.BalanceStrategyRoundRobin
	case "range":
		config.Consumer.Group.Rebalance.Strategy = sarama.BalanceStrategyRange
	default:
		log.Panicf("Unrecognized consumer group partition assignor: %s", assignor)
	}

	if oldest {
		config.Consumer.Offsets.Initial = sarama.OffsetOldest
	}
	kafkaClient := &KafkaClient{
		topics: strings.Split(topics, ","),
	}
	kafkaClient.consumerGroup, err = sarama.NewConsumerGroup(strings.Split(brokers, ","), groupID, config)
	if err != nil {
		log.Panic(err)
	}
	return kafkaClient
}

func (k *KafkaClient) Consume(handler Handler) {
	keepRunning := true
	log.Println("Starting a new Sarama consumer")

	/**
	 * Setup a new Sarama consumer group
	 */
	consumer := Consumer{
		ready:   make(chan bool),
		handler: handler,
	}

	ctx, cancel := context.WithCancel(context.Background())

	consumptionIsPaused := false
	wg := &sync.WaitGroup{}
	wg.Add(1)
	go func() {
		defer wg.Done()
		for {
			// `Consume` should be called inside an infinite loop, when a
			// server-side rebalance happens, the consumer session will need to be
			// recreated to get the new claims
			if err := k.consumerGroup.Consume(ctx, k.topics, &consumer); err != nil {
				log.Panicf("Error from consumer: %v", err)
			}
			// check if context was cancelled, signaling that the consumer should stop
			if ctx.Err() != nil {
				return
			}
			consumer.ready = make(chan bool)
		}
	}()

	<-consumer.ready // Await till the consumer has been set up
	log.Println("Sarama consumer up and running!...")

	sigusr1 := make(chan os.Signal, 1)
	signal.Notify(sigusr1, syscall.SIGUSR1)

	sigterm := make(chan os.Signal, 1)
	signal.Notify(sigterm, syscall.SIGINT, syscall.SIGTERM)

	for keepRunning {
		select {
		case <-ctx.Done():
			log.Println("terminating: context cancelled")
			keepRunning = false
		case <-sigterm:
			log.Println("terminating: via signal")
			keepRunning = false
		case <-sigusr1:
			toggleConsumptionFlow(k.consumerGroup, &consumptionIsPaused)
		}
	}
	cancel()
	wg.Wait()
	if err := k.consumerGroup.Close(); err != nil {
		log.Panicf("Error closing client: %v", err)
	}
}

func toggleConsumptionFlow(client sarama.ConsumerGroup, isPaused *bool) {
	if *isPaused {
		client.ResumeAll()
		log.Println("Resuming consumption")
	} else {
		client.PauseAll()
		log.Println("Pausing consumption")
	}

	*isPaused = !*isPaused
}

// Consumer represents a Sarama consumer group consumer
type Consumer struct {
	ready   chan bool
	handler Handler
}

// Setup is run at the beginning of a new session, before ConsumeClaim
func (consumer *Consumer) Setup(sarama.ConsumerGroupSession) error {
	// Mark the consumer as ready
	close(consumer.ready)
	return nil
}

// Cleanup is run at the end of a session, once all ConsumeClaim goroutines have exited
func (consumer *Consumer) Cleanup(sarama.ConsumerGroupSession) error {
	return nil
}

// ConsumeClaim must start a consumer loop of ConsumerGroupClaim's Messages().
func (consumer *Consumer) ConsumeClaim(session sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
	// NOTE:
	// Do not move the code below to a goroutine.
	// The `ConsumeClaim` itself is called within a goroutine, see:
	// https://github.com/Shopify/sarama/blob/main/consumer_group.go#L27-L29
	for {
		select {
		case message := <-claim.Messages():
			log.Printf("Message claimed: value = %s, timestamp = %v, topic = %s", string(message.Value), message.Timestamp, message.Topic)
			err := consumer.handler.Handler(message)
			if err != nil {
				log.Println(err)
			} else {
				session.MarkMessage(message, "")
			}

		// Should return when `session.Context()` is done.
		// If not, will raise `ErrRebalanceInProgress` or `read tcp <ip>:<port>: i/o timeout` when kafka rebalance. see:
		// https://github.com/Shopify/sarama/issues/1192
		case <-session.Context().Done():
			return nil
		}
	}
}

type Handler interface {
	Handler(message *sarama.ConsumerMessage) error
}
```

```go
// producer.go
package kafka_demo

import (
	"github.com/Shopify/sarama"
	"strings"
	"time"
)

type kafkaSyncProducer struct {
	Topic        string
	syncProducer sarama.SyncProducer
}

func NewKafkaSyncProducer(brokers string, topic string) (*kafkaSyncProducer, error) {
	pr, err := sarama.NewSyncProducer(strings.Split(brokers, ","), nil)
	return &kafkaSyncProducer{
		Topic:        topic,
		syncProducer: pr,
	}, err
}

func (k *kafkaSyncProducer) SendMessages(messages [][]byte) (err error) {
	producerMessages := make([]*sarama.ProducerMessage, 0, 1)
	for _, msg := range messages {
		producerMessages = append(producerMessages, &sarama.ProducerMessage{
			Topic:     k.Topic,
			Key:       nil,
			Value:     sarama.ByteEncoder(msg),
			Headers:   nil,
			Metadata:  nil,
			Offset:    0,
			Partition: 0,
			Timestamp: time.Time{},
		})
	}
	if len(producerMessages) > 0 {
		err = k.syncProducer.SendMessages(producerMessages)
	}
	return
}

func (k *kafkaSyncProducer) SendMessage(message []byte) (partition int32, offset int64, err error) {
	partition, offset, err = k.syncProducer.SendMessage(&sarama.ProducerMessage{
		Topic:     k.Topic,
		Key:       nil,
		Value:     sarama.ByteEncoder(message),
		Headers:   nil,
		Metadata:  nil,
		Offset:    0,
		Partition: 0,
		Timestamp: time.Time{},
	})
	return
}
```

```go
// size.go
package kafka_demo

import (
	"fmt"
	"math/rand"
	"time"
	"unsafe"
)

//
func newMessage() {
	//fmt.Println(unsafe.Sizeof([128]int64{}))
	msg1Kb := new1KbMessage()
	fmt.Printf("1KB: size = %d byte\n", unsafe.Sizeof(msg1Kb))

	msg1Mb := new100KbMessage()
	fmt.Printf("100KB: size = %d byte\n", unsafe.Sizeof(msg1Mb))
}

func new1KbMessage() [1024]byte {
	v := [1024]byte{}
	return v
}

func new100KbMessage() [102400]byte {
	v := [102400]byte{}
	for i := 0; i < 102400; i++ {
		rand.Seed(time.Now().UnixNano())
		v[i] = byte(rand.Intn(100))
	}
	return v
}
```

```go
# producer_test.go
package kafka_demo

import (
	"strconv"
	"testing"
)

func newDefaultProducer() *kafkaSyncProducer {
	brokers := "kafka-master-0.kafka-master-headless.ps.svc.cluster.local:9092"
	topics := "test"
	ps, _ := NewKafkaSyncProducer(brokers, topics)
	return ps
}

func TestSyncProducer(t *testing.T) {
	brokers := "kafka-master-0.kafka-master-headless.ps.svc.cluster.local:9092"
	topics := "test"
	ps, _ := NewKafkaSyncProducer(brokers, topics)
	for i := 0; i < 16; i++ {
		val := []byte(strconv.FormatInt(int64(i), 10))
		_, _, err := ps.SendMessage(val)
		if err != nil {
			t.Error(err)
		} else {
			//t.Logf("produce Message: value = %s, partition = %v, offset = %v", string(val), partition, offset)
		}
	}
}

func TestSyncProducer1Kb(t *testing.T) {
	producer := newDefaultProducer()
	m := new1KbMessage()
	counter := 1024 * 50
	for i := 0; i < counter; i++ {
		_, _, err := producer.SendMessage(m[:])
		if err != nil {
			t.Error(err)
		}
	}
	producer.syncProducer.Close()
}

func TestSyncProducer500KB(t *testing.T) {
	producer := newDefaultProducer()
	m := new100KbMessage()
	counter := 512
	for i := 0; i < counter; i++ {
		_, _, err := producer.SendMessage(m[:])
		if err != nil {
			t.Error(err)
		}
	}
	producer.syncProducer.Close()
}

func TestSyncProducerString(t *testing.T) {
	producer := newDefaultProducer()
	m := []byte(`{"name": "Bob", "age": 20}`)
	counter := 512
	for i := 0; i < counter; i++ {
		_, _, err := producer.SendMessage(m[:])
		if err != nil {
			t.Error(err)
		}
	}
	producer.syncProducer.Close()
}
```

###### 3.3 100KB消息写入

1. GZIP

```text
# 50MB消息，单条消息100KB
# 写入前
0       00000000000000000000.index
0       00000000000000000000.log
0       00000000000000000000.timeindex
4.0K    leader-epoch-checkpoint
4.0K    partition.metadata

#写入后
8.0K    00000000000000000000.index
43M     00000000000000000000.log
8.0K    00000000000000000000.timeindex
4.0K    leader-epoch-checkpoint
4.0K    partition.metadata

# 50MB消息，单条消息100KB
# 写入前
8.0K    00000000000000000000.index
43M     00000000000000000000.log
8.0K    00000000000000000000.timeindex
4.0K    leader-epoch-checkpoint
4.0K    partition.metadata

# 写入后
12K     00000000000000000000.index
85M     00000000000000000000.log
16K     00000000000000000000.timeindex
4.0K    leader-epoch-checkpoint
4.0K    partition.metadata
```

2. Snappy

```
# 50MB消息，单条消息100KB
# 写入前
28K     00000000000000000000.index
277M    00000000000000000000.log
40K     00000000000000000000.timeindex
4.0K    leader-epoch-checkpoint
4.0K    partition.metadata

# 写入后
32K     00000000000000000000.index
327M    00000000000000000000.log
44K     00000000000000000000.timeindex
4.0K    leader-epoch-checkpoint
4.0K    partition.metadata
```

3. LZ4

```
# 50MB消息，单条消息100KB
# 写入前
32K     00000000000000000000.index
327M    00000000000000000000.log
44K     00000000000000000000.timeindex
4.0K    leader-epoch-checkpoint
4.0K    partition.metadata

# 写入后
36K     00000000000000000000.index
377M    00000000000000000000.log
52K     00000000000000000000.timeindex
4.0K    leader-epoch-checkpoint
4.0K    partition.metadata
```

4. ZSTD
- 测试方式
```text
# 50MB消息，单条消息100KB
# 写入前
24K     00000000000000000000.index
235M    00000000000000000000.log
32K     00000000000000000000.timeindex
4.0K    leader-epoch-checkpoint
4.0K    partition.metadata

# 写入后
28K     00000000000000000000.index
277M    00000000000000000000.log
40K     00000000000000000000.timeindex
4.0K    leader-epoch-checkpoint
4.0K    partition.metadata
```
- 结果
|  压缩算法  |  GZIP  |  Snappy  |  ZSTD  | 
|:------:|:------:|:--------:|:------:|
|  压缩率   |  0.86  |    -     |  0.84  |



