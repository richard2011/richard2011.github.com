---
layout: post
title: 分布式消息队列选型 
categories: 中间件
tags: tracing  
---

{{ page.title }}
================

## 目的：

我们正在构建公司下一代交易系统，MQ在整个构架都起在重要作用。

我们希望这系统能做到以下几点：

1. 消峰（堆积）/ 高吞吐量。
2. 监控日志传输（cep、流计算）。

## 备选方案：
RocketMQ / Kafka / RabbitMQ

## Kafka（RocketMQ）与传统MQ比较:
Kafka与传统MQ最大不同的是
1. 消费会待久化，充分利用顺序（或叫线性）写入性能。
2. 客户端（Producer & Customer）比较重，如维护offset，缓存，批量等。
3. 天生分布式。

## 对比

对比项目 | RocketMQ | Kafka | RibbitMQ |
--------------- | -------------   | ------------- | ------------- | 
开发语言  | Java  | Scala | Erlang |
成熟度 | 开源版存在一定BUG，最后一次更新在大半年前，有商业版ONS |  Apache顶级项目，Linkedin公司支持。最新版本0.8.2.2（2015-10-03发布) | 成熟度高，文档完善，API友好。 |
管理工具 | 命令行，非官方简易admin | 命令行，非官方简易admin | 官方admin, 功能完善 | 
生产消息性能 | 3个broker大概7W msg/sec [注1] | 单个Producer , 786,980 msg/sec (75.1 MB/sec) [注2] 三个Producer, 2,024,032 msg/sec (193.0 MB/sec) | 149,910 msg/sec | 
消费消息性能 | 50W msg/sec | 单个Consumer 940,521 msg/sec (89.7 MB/sec)三个Consumer 2,615,968 msg/sec (249.5 MB/sec) | 64,315  msg/sec | 
生产与消费同时 | 6W msg/sec | 795,064 msg/sec (75.8 MB/sec) [注3] | 32,005 msg/sec [注4] | 
消息堆积 | 亿级 | TB级 | 百万级 |
消息实时性 | push | 0.8后支持push方式（long poll）| push/push |
消息传递可靠性 | 多 Master 多 Slave 模式,异步复制，当Master宕机，会丢失少量消息 | 异步复制，当Master宕机，会丢失少量消息。| 基于AMQP协议，具有较高可靠性。|
消息堆积时性能 | 有无salve会有一定影响 | 得益于顺序写，顺序读。不受堆积影响。 | 堆积用占用大量内存，负载会变高 |
高可用 | nameserver通过HTTP 静态服务器寻址（2分钟更新,broker主-从 | zookeeper自带高可用,broker副本复制 | 应对网络分区较弱 |
水平扩展 | 增加broker（1主1从）| 增加broker | 无 |


 
注：   
1. 与官方数据出入比较大，可能官方使用SAS磁盘或SSD。官方数据为 『RocketMQ单机写入TPS单实例约7万条/秒，单机部署3个Broker，可以跑到最高12万条/秒，消息大小10个字节』。   
2. Producer端将多个小消息合并，批量发向Broker，所以远比RocketMQ高。这是副本数为3，异步的结果。这两个是重要因素，如果是同步，这结果大概减小一半。   
3. Kafka官方设计文档提到Consumer消费应该尽量命中pagecache，Consumer很cheaper，不会受写消费和堆积影响。RocketMQ在这块改为顺序写，随机读，具体看后面与Kafka架构对比。   
4. RabbitQM由于AMQP严格的消费机制，性能符合预期。
## Benchmarking
### 6台机器   
* Intel Xeon E5-2600 2.5 GHz processor with six cores  
* 7200 RPM SATA drives  
* 32GB of RAM  
* 1Gb Ethernet

### 测试场景
#### 生产者吞吐能力
场景一：单个生产者，消息大小256个字节   
场景二：三个生产者，消息大小256个字节
#### 消费者吞吐能力
场景一：单个消费者，消息大小256个字节  
场景二：三个消费者，消息大小256个字节
#### 生产者消费者吞吐能力
场景一：单个生产者，单个消费者，消息大小256个字节   
场景二：三个生产者，三个消费者，消息大小256个字节
#### 消息大小对性能影响
## FAQ
### 1. Kafka 会丢消息（如何减小或快避免）？
* ***producer发送消息到broker***  
配置request.required.acs为1。batch.num.messages（合并小消息，批量发送）改为1，默认为200。（轩辕场景：task_center）
* ***broker消息到consumer***   
自己维护offset（使用low-level API），这个出现机率较小。官方推荐使用hight-level API（做法为保存在内存，定时同步到zookeeper，0.9用java重写consumer api，去掉zookeeper）。
* ***broker***   
0.8.x版本以后，可以通过replica机制保证数据不丢，代价就是需要更多资源，尤其是磁盘资源，kafka当前支持GZip和Snappy压缩，来缓解这个问题。 使用同步复制。   

### 2. Kafka 和 RocketMQ 架构上不同？
* ***RocketMQ 在3.x去掉zookeeper***   
zookeeper在连接大量客户端会有一些问题，而RocketMQ要上阿里云，改为无状态轻量级的name-server.
* **零拷贝**  
零拷贝有二种方式， RocketMQ使用mmap的方式（小块文件传输效率高，比较耗CPU，内存控制比较复杂），Kafka使用sendfile方式（大文件传输效率高，消耗CPU较小，无内存刷新问题，这可以解释为什么阿里说比Kafka刷盘快，Kafka为什么做小消息合并。
* **副本复制方案**   
Kafka 0.8后使用『主备模型』（primary-backup model），即选出一个副本做为leader，让leader按请求到达的顺序处理请求，其他的副本保持和leader同步，并能够在leader失败的时候接替它成为leader。zookeeper来提供选举功能。这跟solrCloud做法差不多一样。例，3个borker,副本数为3，则 broker-1（0 1 2 ）/ （broker-2）0 1 2 / （broker-3）0 1 2，黑色为leader。这情模式当broker-1和broker-2宕机了。整个群都可用。RocketMQ是各个broker不通信，单个broker做主-从。这方案从是比较浪费机器，从的唯一作用就是在当主压力比较大的时候，消费者可以在从消费。如果是主和从同时宕机了呢？
* **分区**   
为了能支持更多的Topic数量，RocketMQ把分区概念去掉，引用 commit log的概念。这导致了变成顺序写，随机读。所以为什么Consumer吞吐量上不了。

### 3. 哪些因素会影响Kafka的吞吐量？
* 消息batchSIze
* 分区数
* 副本策略(async/sync/replicas)
* 充足的内存(write_throughput*30)，磁盘写速度。
* linux 文件句柄（file descriptors）和 socket buffer size。

### 4. Kafka（RocketMQ）能否虚拟机运行?    
主要影响的因素是零拷贝，但现在没有资料能证明在虚拟机不可以用。



## 参考
1. [Kafka](http://kafka.apache.org)
2. [RocketMQ](https://github.com/alibaba/RocketMQ)
3. [RabbitMQ](http://www.rabbitmq.com)
4. [RocketMQ与Kafka对比（18项差异）](https://github.com/alibaba/RocketMQ/wiki/rmq_vs_kafka)
5. [Benchmarking Apache Kafka ](https://engineering.linkedin.com/kafka/benchmarking-apache-kafka-2-million-writes-second-three-cheap-machines)
6. [RabbitMQ Performance Measurements](http://www.rabbitmq.com/blog/2012/04/25/rabbitmq-performance-measurements-part-2/)
7. [分布式发布订阅消息系统 Kafka 架构设计](http://www.oschina.net/translate/kafka-design)



