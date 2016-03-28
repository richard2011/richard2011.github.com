---
layout: post
title: Kafka培训与分享
categories: 中间件
tags: mq  
---

{{ page.title }}
================

Kafka在卷皮已经上线已经有好几个月了，在轩辕架构中作为一个Procesor流量的入口，每天承载着公司的售卖线业务。本次的技术交流会，我想分为两部分：第一部分是Kafka的培训，希望可以作为大家对Kafka入门的快速指南。第二部分是Kafka在卷皮的实践，与大家分享一些使用过程中遇到问题和思考。


Kafka，一个高吞吐的分布式消息系统，源于LinkedIn，现在是apache顶级项目。大部分由Scala编写，一些Kafka用Java编写，共有9个核心的开发者，社区非常活跃。在卷皮主要用于Proxy与Processor之间解耦，为Processor提供负载均衡。线上的版本是0.9.0.1, 6个broker作集群, 机器的配置为32G内存，600G SAS盘，一共有40+ topics, 日均处理消息量为6000W+。

## 基础培训
Kafka设计思路源于一个简单的抽象-日志（The Log）。注意，这里的日志不是指给人看的日志，而是指像BinLog或CommitLog这一类用于程序访问的日志。日志有两个重要的特性：有序和不可变。我想这两个特性对于一个分布式系统是非常重要的，接下来我会通过介绍Kafka，大家可以带着这个问题去了解。

**术语**

Kafka的角色分为生产者（Producers）、代理（broker）、消费者（Consumer），这三者都是显式分布式的。生产者发布关于某个主题（Topic），这句话的意思是，消息以一种物理方式被发送给了作为Broker的服务器（可能是另外一台机器）。若干的消息消费者订阅（subscribe）某个话题，然后生产者所发布的每条消息都会被发送给所有的消费者。每个消费者都属于同一个消费者小组（Consumer Group）, Kafka保证每一条消费只会发给同一个小组的一个消费者，使用消费者小组这个概念就可以实现JMS中队列（queue）或者话题（topic）这两种语义。 Kafka与传统MQ有一个很大的区别是，对于一个话题而言，无论有多少消费者订阅了它，一条消息都只会存储一次，通过偏移量（offset）来控制消费者的消息处理进度。

**Topic vs. Partition vs. Replicas**   

Kafka的存储非常简单。一个话题包括了多个分区（Partition），话题的每个分区对应一个逻辑日志。物理上，一个日志为相同大小的一组分段文件（Segment file）。每次生产者发布消息到一个分区，Broker就将消息追加到最后个段文件中。在0.8.x之前，没有副本（Replicas）的概念，一旦某个Broker宕机了，其上所有分区不可被消费，如果这个broker的硬盘故障数据就会有丢失，所以在0.8.x之后引入了副本的机制, 同一个partition可能会有多个副本，而这时需要在这些副本之间选出一个leader，producer和consumer只与这个leader交互，其它副本作为follower从leader中复制数据，当leader宕机后会重新选举leader,其中复制因子（replication-factor）可以配置，broker失败容忍性是replication-factor - 1，如replication-factor = 2，可容忍1个broker宕机。

**Producer & Consumer**    

Producer默认是循环的策略来向副本Leader发送消息，同时提供三种ACK的级别，分别为-1（所有副本都接收），1（Leader副本接收），0（NO ACK）。最后我们聊一下Comsumer， 首先是Comuser Rebalance，其实只需要记住一句话：Partions的数量决定着Comuser的并行度， 也就说如果某Consumer Group中Consumer数量少于Partition数量，则至少有一个Consumer会消费多个Partition的数据，如果Consumer的数量与Partition数量相同，则正好一个Consumer消费一个Partition的数据。而如果Consumer的数量多于Partition的数量时，会有部分Consumer无法消费该Topic下任何一条消息。

## 一些实践：

**『最少一次』的副作用**    
线上sku-domainloader曾经出现过Comsumer 频繁rebanlance的情况。原因是redis写入太慢导致整个Consumer消费处理过慢，大于session timeout时间，offset没有提交就rebanlance, 这样会导致消息重复消费。Kafka Consumer默认的消息传递语义是『最少一次』，当Consumer的业务执行完之后才提交offse，而且Kafka还采用了批量fetch提交offset最高位（HW，HighWater）的方式，就把这问题更放大。最后我们得出来的结论是，使用Consumer的『最少一次』语义来消息，不应该阻塞kafka client Consumer的进程。   

**消息好像阻塞了？**   

『消息好像阻塞了？』业务开发经常问这个问题。这个问题可以通过kafka-consumer-group.sh来查看消息消费情况，hight-level consumer可能阻塞的情况：1）消息大于fetch.size（默认为1M）2）应用阻塞代码（如，异常没有捕获，可用JMC，jvisualvm查看线程状态）3）consumer rebalance 失败，会看到ConsumerRebalanceFailedException。总结：不应该阻塞kafka client的线程，时刻关注rebalancing的情况。

**Kafka 会丢消息吗？**   
我觉得可以把问题变成『Kafka为了不丢失消息做了哪些事情？』，Kafka Broker 在0.8.x后引入了副本机制，Producer提供了三种ACK的级别，Consumer提供了『最少一次』的消息传递语义，将来也会提供『仅仅一次』的消息传递语义。在卷皮，为了追求吞量，replication-factor = 2，ack = 1。

**消息能有序吗？**
Kafka保证单Partition有序，想想Partition的写入机制就很容易理解了。客户端具体做法可以为：1）Producer send指定分区 2）Consumer指定分区消费，需要自己实现fail-over 3)replication-factor设置为N（N为Broker的数量），保证Broker n-1 的宕机容忍性。Linkedin针对这点，还构建一个流式处理系统Apache Samza，由Kafka、Yarn和SamaJob组成，其中Kafka作为消息传递，Yarn作资源调度和配置(指定分区)，Samza的基本处理流程是一个用户任务从一个或多个输入流中读取数据，再输出到一个或多个输出流中，具体映射到kafka上就是从一个或多个topic读入数据，再写出到另一个或多个topic中去。多个job串联起来就完成了流式的数据处理流程了。

**Topic的Partitions数量权衡**

Kafka官方也说了，不用尝试去测试Kafka能支持多少个Partition。Partitions数量是需要权衡的，可以老虎以下几个方面：   

* partitions决定comsuer并行度。
* partitions 只可以增加，不能减小。
* 每个partition都在ZooKeeper上注册。
* 越多partition越久Leader fail-over时间。

总结：根据应用场景制定partitions，关注rebalancing情况，关注ZooKepper情况。   

**Kafka在未来的轩辕**

Kafka希望成为一个把日志作为服务的组件，日志有两个重要的特征：有序和不变性。我想这两个特点为对轩辕或者对分布式系统来说是很重要的，事件就是某个时间点的状态,事件按时间先后顺序产生（有序），只能新增不能删除或修改（不变性），从这个角度看日志和事件并没有什么区别。kafka在0.10.x(2016 Q2发布)也将支持『仅仅一次』消息传递语义的Consumer以及Kafka Stream，也是期值期待的。



