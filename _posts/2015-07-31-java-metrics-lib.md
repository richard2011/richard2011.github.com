---
layout: post
title: 自己实现 java metrics lib
categories: 监控
tags: metrics   
---

## 目的
1. 在JVM计算TPS和执行时间等指标信息，然后定时上报。
2. 可比较容易集成到其他的基础框架中（如SDK,2.0架构）。
3. JAVA服务是否正常工作(healthcheck)。
4. 其他

## 行业方案
在Java界，只有一个Yammer Codehale作为metrics库，其余就是不成熟的Netfilx Servo，以及JavaSimon, JaMon, Perf4j这些单调的更接近于Profiler，twiiter的Ostrich是基于scala。所以，Cassandra是用Yammer， Spring Boot里也是用Yammer。

Codehale有几种Metrics类型：

* Counter，最单纯的Couter。
* Meter，Counter的基础上还带有Rate，包括从系统启动到现在的平均rate，1分钟，5分钟，15分钟的平均rate，15分钟这种移动rate并不是真的保存15分钟然后求平均值，而是在每隔很一小段时间，按照某个公式(使用Linux里Top的算法)对counter变化进行计算推导出1/5/15分钟的值。
* Histogram，适合于计算请求的执行时间，每次塞进去一个当前的执行时间，算出一堆数值的最小，最大，平均，方差，以及50%，75%, 90%, 95%, 98%, 99%, and 99.9% 都会小于某个值。有几种算法决定存储多少的基本数据 ，比如 SlidingWindowReservoir(固定大小)，SlidingTimeWindowReservoir(固定时间长度)，UniformReservoir(随机采样)
* Timer，等于Meter+Hitogram，既算TPS，也算执行时间。

向后报告的方式包括定期报告的如graphite, console, slf4j log，也有即时报告的如JMX。各个reporter间独立Scheduler。


## 为什么要自己实现java metrics lib 
1. 很多数据并不是我们想要的，白白计算白白浪费性能，Meter经常性计算推导的1/5/15分钟平均TPS值有点浪费，影响traffic， 又比如Hisogram，方差对一般人来说不敏感，”50%，75%, 90%, 95%, 98%, 99%, and 99.9% 都会小于某个值“ 也是太多了，最好可以自行配置需要的数值。
2. Meter里只有一个从服务启动到现在的Counter和Mean Rate，以及推导出来的1/5/15分钟平均值。总Counter其实不能很好的解决服务重启以及Long值达到最大值等情况，其实更多情况需要的是每个报告期内的Counter的变化值和TPS。(当然，启动到现在的Counter和平均Rate可以作为一个辅助数据同时提供)
3. 多种Reporter同时使用的话，ConsoleReporter计算一次值，GraphiteReporter再计算一次值。
4. metrics作为一个基础的组件，对性能要高很高，有必要自己维护。

## 具体内容
1. 实现Counter,Counter和Meter合一，并且提供上一次report到现在的counter变化值和rate。主要是利用AtomicLong做计算。后续可能会实现Histogram，用于计算执行时间。
2. report方式，现在是console, slf4j log，后续可能用增加RocketMQ和上报给zabbix等。
3. Yammer Codehale的healthcheck使用的java ThreadMXBean的findDeadlockedThreads()作为检测依据的。是否满足需求有待进一步确认。
4. 更方便的集成方式，可能实现metrics-spring，方便开发人员进一步集成。

## 如何集成
1. 程序启动的时候注册并启动一个调度。
```
scheduler = new ReportScheduler(MetricRegistry.INSTANCE, new Slf4jReporter());
scheduler.start(10, TimeUnit.SECONDS);
```

2. 在方法加上监控
```
private Counter counter = MetricRegistry.INSTANCE.counter("testName");

public void methodXXXX(){
        counter.inc();
        //..
}
```


## 参考
1. [metrics](https://github.com/dropwizard/metrics)
