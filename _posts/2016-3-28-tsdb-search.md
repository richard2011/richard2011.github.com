---
layout: post
title: 时间序列数据库
categories: 中间件
tags: 数据库  
---

{{ page.title }}
================


## 背景
新的监控系统需要存储大量的[metrics](http://metrics20.org)(指标，如CPU、Memory、Network等指标信息)，这样数据的特点都是按时间先后顺序，会做一些简单的过滤，聚合和分组。时间序列数据库就是专门来存储这些数据的，我们希望经能有一个符合我们需求的时间序列数据库。


## 基本概念
定义：时间序列数据库（Times-Series Database，下面简称时序数据库）, 数据格式里包含timestamp字段的数据。一般具有以下特点：

* 结构非常简单，按照关系型数据库的说法，其表结构是这样的：
```
[metricName] [timestamp] [value]
```  
一般都会通过timestamp查询

* 通常会引入tags概念用于数据过滤

* 读写都是提供REST API, 易于跟第三方DashBoard结合，如[Ganfana](http://grafana.org)

* 都是基于key-value数据库构建，列举一下现有开源的项目就可以证明这点，[opentsdb](http://opentsdb.net)（基于hbase），[kairosDB](http://kairosdb.github.io)（基于cassandra），[influxdb](https://influxdata.com)（自己实现[LSM-Tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree)的存储）。

key-value数据库是我们所关注的。
key-value数据库的格式为：
```
Map<RowKey, SortedMap<ColumnKey, ColumnValuealue>>
```  

为什么key-value数据库适合做时间序列数据库?想想下面的结构   
RowKey：metricName、timestampBegin、tags(格式tagkey=ta1gValue,...) 一般会把这两个字段的值进行编码作为唯一标识，timestampBegin   
ColumnKey：timestampOffset，偏移量  
ColumnValue：Value   


如果你了解BigTable，这种就是所谓的『宽行（wide row）』或列族数据库，ColumnKey一般都是有序的行扫描会非常快，ColumnKey的数量可以是很多，如Cassandral列数最大值是Integer.MAX_VALUE（约21亿）。这种Key-Value数据库一般都使用[LSM-Tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree)来储存，这种结构的特点是写很快，读比较慢，不过这些都是可以优化的。


## 选型之痛
有了上面的认识，相信大家已经知道构建一个时序数据库的要点，让我们聊一下现有的开源时序数据库。InfluxDB（最新版0.11.0）不成熟，因为群集和高可用还在是Beta状态，这正是我们所关注的，0.10.0后实现了自己存储引擎（TSM engine），也有自己的WEBUI，野心不小。我们也没有选择OpenTSDB，因为Hbase和Hadoop对我们来说是个庞然大物。KairosDB似乎是个好选择，我们进行了源码的编译和测试，发现它在性能和稳定性上在问题，对于Cassandra的支持只是很旧的版本，而且代码已经很久没人维护，内嵌WEBUI我们认为是个很鸡肋的功能。   
要不说一下我们所希望的： 
  
* 最好是基于Cassandra[1]，因为一方面Cassandra可能是Key-Value中对于Java系来说把控力度较高,另一方面Cassandra可能会成为轩辕的Event Store, 所以Cassandra具有战略意义。
* 高效的写入，虽然大多数开源时间序列数据库都支持Rest API。但是我们不想再搭一个[HAProxy](http://www.haproxy.org)或者[LVS](https://github.com/alibaba/LVS)作负载均衡，还有我们想要性能更高的序列化方案，一个简单直接的方案就是接入Kafka。
* 模块化，大多数开源时序数据库都是All-IN-ONE，我们希望『总是可写的』的，读写可以分开发布。
* 更好的聚合功能，这是大多数开源时序数据库所没有的。

最后我们发现了[heroic](http://spotify.github.io/heroic/)，与我们的想法非常吻合，heroic使用JDK 8开发，[Google Dapper](http://google.github.io/dagger/)作为IOC框架，Cassandra作为Metric存储，[ElasticSearch](https://www.elastic.co)作为MetaData作索引，Kafka作为数据传输，经过对其测试和源码的走读，我们最终选择了heroic作为我们时序数据库。


---

[1]. Cassandra 是高度可伸缩的、最终一致的、分布式的结构化key-value存储方案，构架源于Google BigTable的数据模型与[Amazon Dynamo](http://s3.amazonaws.com/AllThingsDistributed/sosp/amazon-dynamo-sosp2007.pdf)的完全分布式的架构。2007年由facebook开发，2009成为apache项目。


