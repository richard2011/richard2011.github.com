---
layout: post
title: 基于flume实现数据采集传输组件
categories: 监控
tags: flume
---


## 目的
会有很多地方需要数据的采集传输，源可能有日志文件，各类mq，resetful api，自定义协议等。存储可能有elasticsearch,solr,mysql,hdfs,hbase等，需要方便地扩展源和存储。为了保护存储系统，还需要先缓冲一下然后批量同步/异步写到存储系统。最好还有一些简单的监控。业界方案是有logstash和flume。logstash使用jruby开发，与elasticsearch集成比较好。flume是apache 顶级项目，在高可用（可靠性）方面做得比较好，使用JAVA开发，国内用也比较多, 扩展性也比较好。基于以上原因，有必要维护这样的一个组件。

## flume 架构
1. 一个agent以下组件：source(源) --> channel（缓冲） --> sink（写到存储系统/传递到下一个agent），每个组件可以自由组合，可以自由扩展。
2. 每个agent也可以组合成多个.主要用于负载均衡（LoadBalancing）和 故障转移（Failover）。一般这样组合，source_agent（读）--> collector_agent(写)。

## 扩展和改进
1. 编译flume源码，作二次开发。
2. 扩展RocketMQ和RabbitMQ的source。
3. 把flume的监控接入zabbix。


## 参考 
1. [apach flume](http://flume.apache.org)
2. [基于Flume的美团日志收集系统架构和设计](http://tech.meituan.com/mt-log-system-arch.html)