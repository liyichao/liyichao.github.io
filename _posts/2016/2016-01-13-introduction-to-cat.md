---
layout: post
title: "CAT 产品研究"
category:
tags: []
---
{% include JB/setup %}


# Overview

[CAT](https://github.com/dianping/cat) 是 [吴其敏](https://github.com/qmwu2000) 主导设计的基于 Java 开发的实时应用监控平台。他之前在 eBay 工作超过 10 年，对于 eBay 的 CAL 系统有深入了解， CAL 是 CAT 系统当初的原型。

# 目标

CAT 设计目标如下：

* 故障诊断
  * 跨越边界访问(Across Boundary Activity)：故障最可能出现的地方
    * 跨网络：URL，RPC，SQL，Cache
    * 跨角色：浏览器/网络/应用服务器，Nginx/tornado，Controller/Model/View
    * 跨语言：Java call Scala
    * 跨所有权：公共组件，规则引擎
  * 状态记录
    * 系统状态：CPU，Memory，Network
    * 业务状态：参数，结果
* 业务统计
  * 计数/计时：Page，Service，Cache，Database
  * 分布：浏览器，IP，登录状态，缓存命中
  * A/B 测试，性能测试
* 系统优化：响应时间，依赖耦合，嵌套调用
* 容量规划：数据库，应用服务器

如果故障不是在系统边界，那么通过单元测试或 sentry 这种异常工具可以发现故障的原因并修复。

我们先介绍数据模型，然后根据 CAT 提供的 WEB UI 了解其功能。以下描述中，『应该』表示是我觉得应该，CAT 可能并没有这个功能。

# CAT 数据模型

CAT 监控系统将每次 URL、Service 的请求内部执行情况都封装为一个完整的消息树、消息树可能包括 Transaction、Event、Heartbeat、Metric 和 Trace 信息

+  **Transaction**	适合记录跨越系统边界的程序访问行为,比如远程调用，数据库调用，也适合执行时间较长的业务逻辑监控，Transaction用来记录一段代码的执行时间和次数。
+  **Event**	   用来记录一件事发生的次数，比如记录系统异常，它和transaction相比缺少了时间的统计，开销比transaction要小。
+  **Heartbeat**	表示程序内定期产生的统计信息, 如CPU%, MEM%, 连接池状态, 系统负载等。
+  **Metric**	  用于记录业务指标、指标可能包含对一个指标记录次数、记录平均值、记录总和，业务指标最低统计粒度为1分钟。
+  **Trace**     用于记录基本的trace信息，类似于log4j的info信息，这些信息仅用于查看一些相关信息

业务完成埋点后，会产生 CAT 的原始监控信息，即 Logview。

![image](/assets/img/cat/text_view.jpeg)

Transaction 和 Event 都是两级分类。Transaction 可以嵌套，Event 就是单个节点，不可嵌套，Trasaction 和 Event 都可以关联 Key-Value，Transaction 有成功和失败状态和响应时间。它还有另一种展现形式，可以明显看出整个请求耗时在哪。

![image](/assets/img/cat/logview.jpeg)

说到 CAT，就要提到 Google 的 [Dapper](http://static.googleusercontent.com/media/research.google.com/en//archive/papers/dapper-2010-1.pdf) （[zipkin](https://github.com/openzipkin/zipkin) 是其开源实现）。Dapper 追踪 RPC 服务间的调用关系，每次 RPC 调用被称为一个 Span，用户的 URL 请求是根 Span A，假设根 Span 调用了 RPC B、C，则 B、C 为 A 的子 Span，这样，整个请求就被建模为由 A 为根的 Span 树，被称为一个 Trace。每个 span 有 trace_id，span_id，parent_id，所有 span 通过这三个 id 联系起来，一个终端的 URL 请求的所有 span 通过 trace_id 联系起来，dapper 因此获得了端到端的追踪能力。通过对 span 计时，一个 URL 请求在各个系统中所耗的时间都能知道，dapper 也因此被广泛用于性能调优。应用开发者可以在 Span 里记录 Annotation 和 BinaryAnnotation，Annotation 带时间戳，BinaryAnnotation 只是 Key-Value，通过 Annotation，可以实现很多意想不到的功能。例如，可以用来实现 CAT。

首先，从数据采集上看，Span 本身就是二级分类，带的 BinaryAnnotation 就是 Transaction 的 Key-value，而 Event 可以用 Annotation 来实现，Annotation 有时间戳，而二级分类和 Key-value，完全可以通过一定的编码放到 Annotation 的 value 里，而 Heartbeat/Metric，无非是 [Statsd](https://github.com/b/statsd_spec) 里的 Gauges/Timer/Counter，只需要和具体的 Span 关联起来，就可以在前端相关地方展示，Trace 也是 Annotation。其次，就需要基于原始 Trace 做一些统计分析来生成报表，以及在前端展示。

但是，想象不一定都能实现。像 Google 这种超级公司，做出一个通用的 Dapper，有很多其他团队的人会去用好它，例如，日志可以带 trace_id，从而把一个请求的日志都关联起来，并做好很好的展示和报表。对于这种级别的公司来说，日志、数据库、缓存，业务，网络，应该都有比较好的工具去监控和管理，也就不需要 Dapper 去提供这些设计和功能，所以，Dapper 只会去提供调用时间的追踪，而不需要去提供如错误追踪，业务监控等功能。而如果要用 dapper 这种系统去满足监控需求，就需要较多的人力和时间去实现 CAT 现在已经实现的报表。

下面的 Dashboard，TransactionReport、EventReport、ProblemReport、HeartbeatReport、Cross Report 小节，内容基本来自[根据尤勇在【QCon高可用架构群】中的分享内容整理而成的一篇文章](http://www.wmyouxi.com/a/56948.html)，包括图片和文字，为了本文的完整性，把它摘录过来。其中文字稍微有所删减，旧版本的图表也换成新版本的图，并整合了一些其他资源，比如原文里没有讲应用大盘，另外还添加了少量我个人的理解。

# Dashboard

综合整个系统的一些大盘。

业务大盘：核心业务都会定义自己的业务指标，这不需要太多，主要用于24小时值班监控，实时发现业务指标问题，图中一个是当前的实际值，一个是基准值，基准值是根据历史趋势计算的预测值。这个相当于每个业务有一个 Dashboard，上面有多个图表，图表里显示的指标是业务指标。把业务指标单独抽离并显示，足见其重要性。![image](/assets/img/cat/业务大盘.jpeg)

报错大盘：所有应用的 topN 的报错大盘，下图是一个出现故障的图。应用的单位是一个可部署的项目。Transaction **应该**有属性表明它是什么应用，这样就可以把一个应用的报错聚合起来。图中显示的是每分钟的报错排行，比 sentry 更适合于故障时查看什么应用出了问题。每个项**应该**可点击，进入报错的 Transaction，查询具体原因。![image](/assets/img/cat/报错大盘.jpeg)



数据库大盘：

  * 实时知道数据库访问情况的大盘。如何确定存在问题，是根据实时的数据再加一些配置的访问规则。
  * 这里不要用 DB 服务端性能采集的数据（比如io，load，qps等），要用应用程序访问这个 database 得出的响应时间、错误、访问量的数据，这里称之为端到端的数据。
  * 后面的闪电符号是一个 url link，这边可以直接跳转到运维的自动化平台上，做 database 故障降级处理。
  * 每一项**应该**可点击，进入那些访问数据库出问题的 Transaction
  * **应该**把服务端的指标和信息也放到一起展示
  * **应该** 是按库聚合的展示。

![image](/assets/img/cat/数据库大盘.jpeg)

网络大盘：这里采集了核心接入层网络交换机的一些信息，将一些状态定制到监控大盘上，主要的采集指标包括：进入口流量、丢包、错包等。

![image](/assets/img/cat/网络大盘.jpeg)

缓存大盘：与数据库大盘类似

应用大盘：每个应用 OK 不 OK。从图中看，应该是业务情况，一个业务包含多个项目，显示的是每个项目 ok 不 ok。

![image](/assets/img/cat/应用大盘.jpeg)

>什么东西才可以作为一个大盘：这里需要看公司整体运维的故障情况，TopN以及业务指标应该属于通用，数据库和网络大盘是点评在实际经验中经常容易出故障的地方，所以做成大盘这样比较直观的形式，用于发现问题。比如平时因为发布引起故障比较多，他们也做了一个发布大盘，实时监控线上的发布情况。

# Report

Logview 太多，所以有基于 Logview 的报表，这些报表反映了作者对监控的理解，是 CAT 的亮点。

## TransactionReport

![image](/assets/img/cat/transaction_report.png)

这个是 TransactionReport 的一个视图，最顶层可以切换项目，一个项目定义为独立的可部署单元，报表以小时为单位（但是是实时的，最新的 Transaction 信息会在当前小时的报表中显示）。再下一行是机器列表，有一个 All 以及具体的机器，可以看所有机器，也可以看单个机器，如果某台机器出现问题，则可以很容易看到。

![image](/assets/img/cat/transaction_second_type.png)

这里展示了一个Transaction的第一层分类的视图，可以知道这段时间里面一个分类运行的次数，平均响应时间，延迟，95线以及99.9线。95line表示95%的请求的响应时间比这个值要小。

TransactionReport 主要用来做性能优化。看到这个数据，应该想这样的调用量是否合理，这样的耗时有没有办法优化。点击最左边的 show，可以看到响应时间的实时图表。现在时间是 33 分 29 秒，所以图中平均响应时间只有到 32 分钟的数据。![image](/assets/img/cat/transaction_show.png)

## EventReport

和 TransactionReport 类似，可以根据项目，ip 进行导航，但是没有响应时间的概念。主要是帮业务做一些业务统计，如看每天用户的登录次数或者代码里三个分支的执行次数等。

## ProblemReport

ProblemReport 把一些有问题的 Transaction 聚合成一张报表，这些 Transaction 包括：

* 程序里抛出的异常
* URL 访问出错及慢
* 数据库访问出错及慢
* 缓存访问出错及慢
* 服务访问出错及慢

![image](/assets/img/cat/problem_report.jpeg)

上述报表是 tuangou-web 项目一段时间内的各种问题，点击左边的 show 可以看到错误在各 IP 的分布饼图，以及按错误数按时间的分布。

![image](/assets/img/cat/problem_distribute.jpeg)

点击右边 Looooooog 可以看到原始的出错 logview。这个 logview 包含这个请求的整个执行情况，包括 RPC 参数等，很像一个分布式 sentry。

![image](/assets/img/cat/problem_detail.jpeg)

业务开发需要 fix ProbelmReport 中一些比较严重的问题。如 SQL 慢响应很多（通过索引或缓存），服务慢响应（如何优化），调用下游错误（推动下游服务端解决），异常等。

## Heartbeat Report

这里主要是 CAT 客户端定期一分钟向服务端汇报当前运行时候的一些状态，几个比较常用的指标有：

* 系统内存、load
* GC 信息，GC 时间及数量，JAVA 内存状态
* 一些线程数，如 HTTP，RPC 框架线程等

![image](/assets/img/cat/heartbeat.png)

这个报表里也可以展示应用自己打的指标。

## Cross Report

这个报表主要是解决在 RPC 中的一些上下游调用链路的问题，统计粒度支持项目、具体某一 IP、具体的服务方法。

![image](/assets/img/cat/cross_report.png)

比如上图是当前项目 xx-web 调用了 4 个服务，调用每个服务的响应时间，访问量以及错误量。点击某个项目进去可以知道具体调用此服务的具体每台机器的情况。

![image](/assets/img/cat/cross_report_detail.jpeg)

点击 ALL 或者单台机器可以知道调用具体此服务某个方法的耗时情况。

![image](/assets/img/cat/cross_report_detail_method.jpeg)

这个报表还有很多其他作用，比如做为一个服务端，它可以知道当前多少客户端访问我，可以查询服务端一个方法被多少客户端调用，每个客户端调用的响应以及耗时。

![image](/assets/img/cat/cross_report_server.png)

## Cache Report

分请求类型的一些指标的曲线。QPS，响应时间，错误量，命中率，响应时间，错误量等。请求类型包括 get，Mget，delete 等。

## Database Report

错误量，QPS，响应时间，长响应个数等。分请求类型。因此可以方便看到读写比

![image](/assets/img/cat/database_report.png)

另外，还有一个页面，展示服务端的一些指标，如 LockAndWait、InnoDBInfo 等。


## Dependency

趋势图：

![image](/assets/img/cat/dependency.png)

拓扑图：即服务依赖拓扑图。

![image](/assets/img/cat/dependency_top.png)

## Matrix

一次请求（URL、Service）中的调用链路统计，包括远程调用、sql调用、缓存调用。可以看到一个请求中哪一类型的请求占主要作用。

![image](/assets/img/cat/matrix.png)


# Web

URL 访问趋势：某个 URL 的请求数、成功率、成功延时。可以限定和比较日期，返回码，URL，地区，运营商。

![image](/assets/img/cat/url.png)

URL 访问分布：某个 URL 的返回码，运营商，地区分布。

JS 错误日志。

# App

从手机 APP 收集的信息。

API 访问趋势：其纬度包含命令字（应该是业务名），返回码，网络类型，版本，连接类型（长连接短连接），平台（android or ios），地区，运营商。如下图所示。

![image](/assets/img/cat/api_trend.png)

访问量分布：纬度和访问趋势一样，展示请求量在各个纬度上的分布。如在返回码上的分布。

APP 页面测速：如下图所示。

![image](/assets/img/cat/app_speed.png)

每天报表统计：如下图所示。

![image](/assets/img/cat/app_daily_report.png)

Crash 日志：如下图所示。

![image](/assets/img/cat/app_crash_log.png)

# 离线报表

* Pass 系统消耗
* 报表容量统计
* 全局统计异常：按部门，产品线统计的异常数量
* 服务可用性排行
* 线上容量规划：见下图 ![image](/assets/img/cat/capacity_plan.png)
* 重量访问排行：包括远程调用、sql调用、缓存调用最多排行，见下图 ![image](/assets/img/cat/call_range.png)
* 告警智能分析


# 系统监控

* CDN 监控
* 网络监控
* PAAS 监控。包括网络、CPU、Memory，进程，TCP 等系统指标
* 线上变更
* 告警信息：下面小节重点讲

## 告警

告警类型包含：业务告警、网络告警、系统告警、异常告警、心跳告警、第三方告警、前端告警、App 告警、Web 告警、Zabbix 告警、DB 告警、Transaction 告警，http ping 报警。当前报警无需登录可以看到：

![image](/assets/img/cat/alert.png)

而报警配置需要登录才能配置。和其他 CAT 配置放一起，推测报警配置应该是由运维维护。

![image](/assets/img/cat/config.png)

我们可以从 CAT 的报警里学到以下几点：

* 总结了报警的类型。
* 把所有报警整合到了一起。例如 Zabbix，第三方告警，App 报警。

### 业务告警

首先，进行业务分析，产品线有很多指标，来确定产品是否能满足用户需求，由 DW 负责，然后进行业务监控。业务监控关注最重要的业务指标，目的在于出现线上故障时评价影响面，一般一个产品线的核心业务指标不超过 6 个。比如团购，关键指标是：订单创建数量，交易数量，验券数量。一些定义错误的业务指标，比如XXX接口失败，这其实是一个异常指标，当他大量出现时候，其实XXX正常指标肯定是下降。比如XXX响应时间，这是一个性能指标，不是业务指标，当访问量出问题（比如CDN挂了），响应时间还是正常。

CAT 里项目被组织成产品线。在业务大盘中显示的是一个产品线的业务指标。

### Transaction/Event/异常 告警

对所有应用的 Transaction 的执行次数、响应时间、失败率， Event 的执行次数，异常个数进行报警。数据采集已经由 CAT 做，所以只需要配置下报警。

### 网络/系统/心跳 告警

系统层面的告警。网络：网络流量、丢包率；系统：CPU、Memory 等；

### 数据库告警

数据库访问响应时间，错误率的报警

### HTTP ping 告警

根据指定的网址发送HTTP请求，当返回码不为200时发送警告。这个主要是第三方对我们网址的 ping 监控。


# 结论

## Dashboard

业务大盘。业务大盘其实就是业务指标的图表。例如知乎的业务数据包括创建回答数、提问数、邀请回答数、圆桌数、评论数，发送私信数，各主要页面 PV，UV 等。开发肯定是想看这个数据的，评估故障影响也好，看自己的业务是否正常也罢。要把业务数据单独提出来加以强调，并且全面采集业务数据的指标，然后放到一个地方展示。Grafana 一个不方便的地方是使用频率太低，只有绘图功能，点进去还一大堆图，不够有**归属感**。

报错大盘。所有应用的 TopN。这个在出故障时，所有应用的状况都一目了然。如果点击能查看具体为什么错，那就是一个比一条条日志更好的查错系统。从 Overview 开始，到具体的异常上下文，都在 CAT 里展示。这个会增加 CAT 的使用频率。

数据库大盘/缓存大盘。从响应时间，请求量，出错数这三个客户端指标来评估某数据库是否正常。这个也是在出错时会有用，可以看下是不是某个库出问题了，DBA 操作时也会看着这个盘来评估影响。

网络大盘。从网络层面，采集交换机进入口流量、丢包、错包 等指标，看网络有没有超负载，或某链路是否断了等。网络工程师最爱。

应用大盘。直观的显示一个业务（包含很多项目）是否正常。把所有业务正常与否放到一个页面显示使得整个公司的业务是否正常一目了然。

## Report

Transaction report。显示 Transaction 的响应时间、失败率、QPS。一个项目的所有 Transaction 被放到一起显示。先显示 Transaction 的统计信息，可以直观的看到项目的整体情况，点击细节可查看类似 zipkin 那种瀑布图，使得它非常适合用于性能调优。比 zipkin 增加了成功和失败状态，使得它具有定位故障的功能。这里有可以只看某机器的 Transaction，从而可以看到该机器是否有问题。

Event report。可以认为是 Statsd 中计数器。只是增加了两级分类，以及成功失败状态。这里的主要亮点是展示形式。

* 一个项目的所有 Event 被直接以列表的形式展示出来，而不是像 grafana 那样，像是怀有敌意一样的漆黑一片。
* 以表格而不是趋势图的形式展示。这样就能一次展示所有 Event 的请求数，失败数，和响应时间。有时并不需要展示一个趋势，而只需要展示当前的状态。

Problem report。把程序里抛出的异常、URL 访问出错及慢、服务访问出错及慢、缓存访问出错及慢、数据库访问出错及慢这些情况对应的 Transaction 放到一起，做统计，做展示。用于查错，其优点是把所有错误都集中到一起了，并且应用的异常不仅像 Sentry 一样有一些本进程的上下文，还有该 URL 请求在其他机器，其他服务的上下文。

Cross report。RPC 调用中的上下游管理。

* 我调用的所有其他服务，调用的 QPS、响应时间、失败次数。表格展示，可以点击，细化到某方法，某机器。如果有随时间变化的趋势图就更好了。
* 谁调用了我。指标同上。

Database report。各项目对数据库的 select 等调用的错误量，QPS，响应时间，长响应个数等。可以看到读写比。数据库服务端的指标也可以拿过来一起展示。

Dependency report。项目依赖的指标趋势图，以及依赖拓扑图。拓扑图可以看到本项目依赖的其他项目的健康状况，有利于查错。

Matrix report。本项目一次请求（URL、Service）中，RPC、SQL、Cache 的占比情况。

State report。CAT 自身的一些数据。

System report。

* CDN 监控。应该是把 CDN 监控相关页面拿过来了。
* 网络监控。应该是各交换机的趋势图。在网络大盘只显示当前值。
* PAAS 监控。应该是该项目使用的容器的相关指标的图
* 线上变更。包含 puppet，workflow，lazyman。其中 [workflow](http://mageedu.blog.51cto.com/4265610/1662875) 是一套流程系统，其核心思想是把线上所有的变更以标准化流程的方式，梳理出来。
* 告警信息。显示所有正在发生的报警。类型包括：业务告警、网络告警、系统告警、异常告警、心跳告警、第三方告警、前端告警、App 告警、Web 告警、Zabbix 告警、DB 告警、Transaction 告警，三方 http ping 报警。这里亮点是整合。不仅是 CAT 本身产生的报警，连 Zabbix，第三方的报警都放到一起显示。使人在一个地方能方便的知道有没有报警。

离线报表。

* 服务可用性排行。
* 重量访问排行。远程调用最多，数据库最多，缓存最多的项目。
* 线上容量规划。主要显示某项目，目前所占用的机器数，访问量，集群 QPS，单机 QPS，错误率，响应时间，机器 load（平均+最大），机器内存。其中集群 QPS 和单机 QPS 可以衡量应用的水平扩展能力，而机器资源和 QPS 及响应时间放到一起展示，有利于规划容量。
* PAAS 系统消耗。展示 cpuUsage + 机器数，cpuUsage 全天，cpuUsage 高峰。应该是评估 PAAS 的容量。
* 报表容量统计。评估各项目的报表所占据的磁盘空间。如果特别大，可能需要和项目组沟通采集上的优化。
* 告警智能分析。

此外，CAT 针对 Web 和 App 定制了一些页面，包含 URL 请求某 URL 的请求量、成功率、延时等趋势图，亮点在于这些图可以做对比图，如 App 的请求，就可以比较 URL 请求在网络类型，版本，平台（ios or Android），地区，运营商等的分布情况。还有每天报表统计。这个也是亮点。很多人都想有个关于自己应用的请求的一些情况的每日报表。

# 一些可能的坑

## Java 客户端
* Message id 的生成有文件 IO，如果没有持久化，应用重启，id 会重复，即使有持久化，如果不是 graceful restart，也会重复，这些，在 docker 环境下如何解决。重复的影响有多大。
* MessageTree 可能会占用大量内存。cat 有消息截断并立即发送，但还是有点担心。
* IO。传消息的 IO 影响有多大
* CPU。CAT 客户端使用了不少线程，不像 JAVA，python 的线程有 GIL，python 客户端如何实现那些线程的功能。

# 链接

* [吴其敏 InfoQ 演讲](http://www.infoq.com/cn/presentations/public-comments-monitoring-platform-analyse)
* [尤勇在【QCon高可用架构群】的分享](http://www.wmyouxi.com/a/56948.html)
* [大众点评运维架构图文详解](http://mageedu.blog.51cto.com/4265610/1662875)
* cat 依赖 maven 3.2.3，官方已没有，使用以下[下载地址](https://repo1.maven.org/maven2/org/apache/maven/apache-maven/3.2.3/)
