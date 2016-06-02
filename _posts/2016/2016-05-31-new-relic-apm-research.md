---
layout: post
title: "new-relic APM 研究"
category:
tags: []
---
{% include JB/setup %}

# APM 需求分析

用户需要什么？

## 日常运维

* 当应用总体的（加上某些重要接口）错误率或响应时间达到阈值时给报警，并调查原因。
 * 查看自己的应用哪些接口响应时间高了，是自己的原因还是别人的原因，那个接口依赖的其他服务响应时间有没变化，抓几个请求看看整个处理过程
 * 哪些接口错了，主要错在哪些错误类型，那个类型的异常栈/本地变量/请求相关的参数，这个错误所在的整个请求上下文是怎样的，这个错误影响了多大，导致了多少 5xx，影响了多少用户

* 性能优化。
 * 查看外部调用次数和在响应时间的占比。努力减少调用次数、催促下游改善响应时间
 * 查看每个请求的执行过程，找到优化的点




# 首页

[首页](/assets/img/newrelic/All_applications.png)。

* 导航栏 Menu 很整齐，悬浮上去会高亮，有反馈，用户心理比较舒服；第一个用来切产品
* 专门的文档站点；API 文档用户可以交互 （API explorer）；文档有搜索，分门别类，特别丰富
* APM 菜单包括：applications、service maps、key transactions、alerts
* app 可以有 label，方便搜索
* app 列表每一项包含：颜色条（应该是表明健康不健康）, Name, End User, Page Views, App server, Throughput, Error%,
settings (跳往 alert 和 app 相关配置)
* 列表中每一项都可以有链接：Error% 链接到错误分析。
* Recent Events

# Error

## 启发

sentry、newrelic 的错误给的启发：

* 先把所有错误收集起来。
* 错误数据的 slice：错误类型，错误所在的 transaction，错误的其他 key value pair (server)，请求的相关属性（status, browser, device, url, query)
* 异常栈及 locals，replay。
* 请求 ID.




## 界面

[错误](/assets/img/newrelic/Application_Errors.png)。

* 可定制 grouped by 属性
  * 左边有 filter 和 group by.
  * currently grouped by.
  * error attributes: error class, error message, host, transaction name; custom attributes; request header attributes; other request attributes (request method).

* Top 5 errors by `grouped by`。图表之间空白的地方，悬浮上去，会显示 View query 和 View in insights。
* Top 5 transaction by error count
* Error traces 列表。列表项是 Timestamp, transaction name, error class, error message.




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
