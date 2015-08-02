---
layout: post
title: 分布式服务跟踪系统 
categories: Java
tags: tracing  
---

{{ page.title }}
================

## 目的：

一个大型的网站将会由多个分布式系统组成，用户通过浏览器或移动客户端的每一个HTTP请求到达应用服务器后，会经过很多个业务系统或系统组件。但是这些分散的数据对于问题排查，或是流程优化都帮助有限。

我们希望这系统能做到以下几点：

1. 帮助开发快速定位问题，如程序出错时打印traceId，并通过traceId能看到整个调用链。
2.  收集每一处性能数据，并根据策略对各子系统做流控或降级。
3.  这些数据能在决策支持层面发挥作用，如服务器扩容。

## 业界方案：
对于链路监控这块，业界的论文当属[Dapper](http://bigbully.github.io/Dapper-translation/)这篇，它详细的阐述了如何对请求调用链进行跟踪，提出了理论模型，然后它没有具体的代码实现。

Dapper基本原则。

* 低消耗(Low overhead): 分布式在线系统对于性能、资源要求都很严格，Trace组件必须对服务影响足够小，否则一定不会被启用。

* 应用层透明(Application-Level Transparency): 应用层开发者不需要对Trace组件关心，Trace嵌入到基础通用库中，提供高稳定性，而如果Trace需要应用开发者配合，那可能会引入额外的bug导致Trace系统更加脆弱。

* 扩展性(Scalability): Google的服务在线集群数量可想而知，一次检索就可能跨越上百台甚至成千台机器，因此这个Trace Infrastructure的扩展性也很重要。快速的数据分析(Analysis Quickly): 数据快速导出和分析，能够快速的定位系统的异常，及时应对，提升系统的稳定性。


阿里鹰眼(EagleEye) | witter zipkin | 京东hydra | 新浪微博 watchman |
--------------- | -------------   | ------------- | ------------- | 
入口为web页面，无线和开放平台API，所有中间件都是自己，埋点有先生优势。  | zipkin已经被集成到一些基础的框架和类库之中。GC logs也被作为数据来源的一项  | 入口为Dubbo，仅仅实现dubbo调用跟踪，扩展dubbo filter + ThreadLocal | 利用字节码增强方式（javaagent）+ ThreadLocal。对非RPC场景。对RPC场景,使用Motan（自身PRC框架）filter + ThreadLocal（跟京东类似）| 
hash(traceId)，比如采样率被设置为10时，只有hash(traceId) mod 10的值等于0的日志才会输出。| 自适应采样 | 固定时间段内固定跟踪数量的采样，如1秒内超过100，只取10% | 阀门策略，顾名思义，就像一个阀门，用来控制流量的大小，或是开启/关闭。 |
异步log,日志收集agent(应该实现类似flume) | Scribe作为日志收集传输 | 用Java并发库中ArrayBlockingQueue，然后通过dubbo 协议传输 | 异步log，Scribe作为日志收集传输 |
基于数据库：用traceId关联查询，基于Hbase：TraceId做rowkey,实时性强。基于HDFS:顺序存储，后续MapReduce汇总。| 最先使用的是Cassandra，后来加入了Redis, HBase, MySQL, PostgreSQL, SQLite, and H2等存储系统的支持 | 支持Mysql和Hbase | HBASE、HDFS、Storm |

总结：链路监控的核心是：用一个全局的 ID将分布式请求串接起来，在JVM内部通过ThreadLocal传递，在跨JVM调用的时候，通过中间件（http header, framework）将全局ID传递出去，这样就可以将分布式调用串联起来。关于埋点，京东hydra是最简单方案，阿里鹰眼是最强大是也是最复杂的，twriiter zipkin GC logs 收集和新浪微博 watchman的非rpc jvm跟踪是个亮点。关于存储和分析，都采用了HBASE.

## 概念
**trace**:一次服务调用追踪链路。

**span**:追踪服务调基本结构，一次完整的调用（一来一回）称为span。一个span 多个anntion，多个span组成一次trace。

**annotation**:在span中的标注点，记录整个span时间段内发生的事件（包括ip、port、 timestamp）。

**binaryAnnotation**:属于Annotation一种类型和普通Annotation区别，其实就是一个k-v对，记录任何跟踪系统想记录的信息，比如服务调用中的异常信息，重要的业务信息等等。

**cs**: Client Send, span的开始。

**sr**: Server Receive, 接受请求并开始处理。

**ss**: Server Send, 处理完成并返回

**cr**: Client Receive, 客户端接收，span结束。

总时间 = cr(timestamp) - cs(timestamp)

服务调用时间 = ss(timestamp) - sr(timestamp)


## 参考
1. [Dapper，大规模分布式系统的跟踪系统](http://bigbully.github.io/Dapper-translation/)
2. [唯品会Microscope——大规模分布式系统的跟踪、监控、告警平台](http://blog.csdn.net/alex19881006/article/details/24381109)
3. [Zipkin](https://twitter.github.io/zipkin/)
4. [阿里鹰眼](http://wenku.baidu.com/link?url=xsorjRmT7vuIedegixzLF5uC4q5KooXqC-ePnPRKm1eunUDfnjU3vDlPkZqWgHbSCUJUIUivM8FnVCsMZcde0xTxCUu9t0DVFhDKLJdBQye)
5. [Watchman系统](http://www.infoq.com/cn/articles/weibo-watchman)



