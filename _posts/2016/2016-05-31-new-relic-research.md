---
layout: post
title: "new-relic 产品研究"
category:
tags: []
---
{% include JB/setup %}

# 首页

[首页](/assets/img/newrelic/All_applications.png)

# 随感

* Transaction breakdown：对应用内部函数调用的分解。每个段统计的指标有 %time，average calls per transaction，平均时间
* X-Ray sessions：主动触发更详细的 transaction traces + thread profile
* thread profile：类似 strace，用来找到应用程序的性能瓶颈。仔细的追踪程序的执行情况，最细致的分析。

* 接口慢了，需要追查。那么需要把接口处理时间分解。首先客户端的响应时间分解为排队时间和处理时间，处理时间分解为外部调用和内部调用。
外部调用包括 URL 请求，SQL，Redis，RPC。内部调用是指内部函数处理时间，可以通过 profile，或者开发者埋点的方式分解。外部调用分解
主要是 SQL，Redis 的分解。
  * SQL 按 SQL 语句分解：一次接口请求调用该语句的次数、每个语句在整个接口请求中的响应时间占比，该语句平均
响应时间等。每种语句，通过 sql explain 查询该语句响应时间慢的原因，另外，记录下该语句对应的实际实例化时的参数，让开发者可以更进一步
的了解情况。SQL 语句的 stack trace，让开发者可以知道哪段代码调用了 SQL。
  * Redis 也一样。

* 直方图分布
* 和 ticket 系统整合
* 针对应用的报警策略
* 通过标签页切换不同的图表
* [通过配置加打点](https://docs.newrelic.com/docs/agents/python-agent/custom-instrumentation/python-custom-instrumentation-config-file)
