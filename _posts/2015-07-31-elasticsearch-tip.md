---
layout: post
title: elasticsearch优化小记 
---

{{ page.title }}
================

## 目的：

一个大型的网站将会由多个分布式系统组成，用户通过浏览器或移动客户端的每一个HTTP请求到达应用服务器后，会经过很多个业务系统或系统组件。但是这些分散的数据对于问题排查，或是流程优化都帮助有限。

我们希望这系统能做到以下几点：

1. 帮助开发快速定位问题，如程序出错时打印traceId，并通过traceId能看到整个调用链。
2. 收集每一处性能数据，并根据策略对各子系统做流控或降级。
3. 这些数据能在决策支持层面发挥作用，如服务器扩容。

## 业界方案：
对于链路监控这块，业界的论文当属[Dapper](http://bigbully.github.io/Dapper-translation/)这篇，它详细的阐述了如何对请求调用链进行跟踪，提出了理论模型，然后它没有具体的代码实现。

Dapper基本原则。

* 低消耗(Low overhead): 分布式在线系统对于性能、资源要求都很严格，Trace组件必须对服务影响足够小，否则一定不会被启用。

* 应用层透明(Application-Level Transparency): 应用层开发者不需要对Trace组件关心，Trace嵌入到基础通用库中，提供高稳定性，而如果Trace需要应用开发者配合，那可能会引入额外的bug导致Trace系统更加脆弱。

* 扩展性(Scalability): Google的服务在线集群数量可想而知，一次检索就可能跨越上百台甚至成千台机器，因此这个Trace Infrastructure的扩展性也很重要。
快速的数据分析(Analysis Quickly): 数据快速导出和分析，能够快速的定位系统的异常，及时应对，提升系统的稳定性。


阿里鹰眼(EagleEye) | witter zipkin | 京东hydra | 新浪微博 watchman |
--------------- | -------------   | ------------- | ------------- | 
入口为web页面，无线和开放平台API，所有中间件都是自己，埋点有先生优势。  | zipkin已经被集成到一些基础的框架和类库之中。GC logs也被作为数据来源的一项  | 入口为Dubbo，仅仅实现dubbo调用跟踪，扩展dubbo filter + ThreadLocal | 利用字节码增强方式（javaagent）+ ThreadLocal。对非RPC场景。 | 对RPC场景,使用Motan（自身PRC框架）filter + ThreadLocal（跟京东类似）|




