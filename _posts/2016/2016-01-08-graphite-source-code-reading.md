---
layout: post
title: "graphite 源码剖析"
category: 
tags: []
---
{% include JB/setup %}


# Overview

[graphite](https://github.com/graphite-project/graphite-web) 是一个时间序列数据库 (time-series database），它简单稳定，[社区广](http://obfuscurity.com/2015/11/Everybody-Loves-Graphite)，[生态好](http://graphite.readthedocs.org/en/latest/tools.html)，[render API](http://graphite.readthedocs.org/en/latest/render_api.html) 简单易用，支持的函数丰富，并且有 [集群方案](https://grey-boundary.io/the-architecture-of-clustering-graphite/)，这些特点使 graphite 已经成为了时间序列数据库事实上的标杆。

graphite 包括 [三个组件](https://github.com/graphite-project/graphite-web)：

* Graphite-Web, a Django-based web application that renders graphs and dashboards
* The [Carbon](https://github.com/graphite-project/carbon) metric processing daemons
* The [Whisper](https://github.com/graphite-project/whisper) time-series database library

下面直接进入正题，记录下我看 graphite 源码的一些理解。

# carbon-cache

carbon 是用 twisted 网络库实现的，twisted 太复杂，暂时没有精力去了解，但 carbon 如何启动和运行，基本可以推测出来，其代码在 `service.py` 里。carbon 被组织成 `MultiService`，所有 carbon 的 server 都做为一个 twisted 服务被挂到一个根服务，根服务被启动了，所有服务就被启动了。

carbon 主要组件如下：

* Receiver。包括 `MetricLineReceiver`、`MetricPickleReceiver`。监听相应端口，接收、解析指标并发出 metricReceived 事件，事件内容是收到的指标名字和指标点
* Pipeline。metricReceived 事件的监听者是 `run_pipeline`，即运行一个流水线。对于 cabon-cache 来说，流水线的处理工序是类定义有 `plugin_name = 'write'` 的 `Processor`，全局搜索 `plugin_name = 'write'`，知道只有一个 `CacheFeedingProcessor`。该类只是把数据点往一个 MetricCache 单例写，缓存住。
* Writer。代码在 `writer.py`。writer 是一个死循环，把 MetricCache 的内容不断地往磁盘写（一旦写到磁盘，数据点就从 MetricCache 清除了，下次读，要从磁盘读）
* CacheManagementHandler。代码在 `protocols.py`。提供查询 MetricCache 内容的 API 接口。当调用 graphite render api 时，graphite web 会读磁盘文件以得到时间区间内的指标点，但是最新的指标点可能还在 MetricCache，还没写到磁盘，所以 graphite web 还需要查询 MetricCache 的内容，以得到所有数据点。

做为一个良心系统，carbon 自身会打一些关于它自身的指标，可以设置其指标前缀（CARBON_METRIC_PREFIX，默认为 carbon）和指标间隔（CARBON_METRIC_INTERVAL，默认为 1 分钟），其代码在 `instrumentation.py`：

* metricsReceived。收到的指标点数量。相同指标的不同点是单独计算的，所以从这个指标不能知道有多少唯一的指标数（whisper 一个指标一个文件，因此可以通过统计文件数得到唯一的指标数，这个在 whisper 小节会讲），但可以知道每秒的指标量，这个指标可以表明系统负载大小
* cache.queries。来自 graphite-web 对 MetricCache 的请求数
* cache.bulk_queries。来自 graphite-web 对 MetricCache 的批量请求数
* cache.overflow。MetricCache 满了的次数。MetricCache 满了，会停止接收指标.但 MetricCache 的大小小于 `CACHE_SIZE_LOW_WATERMARK` 时重新开始接收指标。如果再满，则又会记一次 overflow。MetricCache 的容量默认为 inf，在这种情况下 overflow 永远为 0.
* cache.queues。在 cache 中的独立指标数。一个指标记 1。
* cache.size。在 cache 中的指标点的个数。一个指标的不同点都被单独计算。

# carbon-relay 和 carbon-aggregator

这两个基本机制和 carbon-cache 一样，这里只讲差异。relay 有以下组件：

* `RelayProcessor`。是 pipeline 的一环，会把数据点传给 `CarbonClientManager.sendDatapoint`
* Router。有多个 destinations，根据一定的规则将指标发送给对应的目标服务器，主要的是 `ConsistentHashingRouter`。
* `CarbonClientFactory`。给服务器发送指标的客户端，维护了一个队列，指标点写入队列，定时（TIME_TO_DEFER_SENDING）将队列的指标点发往对应的服务器

aggregator 有以下步骤：

`rewrite:pre -> aggregate -> rewrite:post -> relay`

rewrite 配置文件为 `rewrite-rules.conf`。rewrite 用来改写指标名字。aggregate 配置文件为 `aggregation-rules.conf`。aggregate 对指标做聚合，生成新指标，并可选的（FORWARD_ALL）将原指标也发往后端服务器

carbon-relay 和 carbon-aggregator 的后端一般是 carbon-cache。这里要推荐下 [carbon-c-relay](https://github.com/grobian/carbon-c-relay)，完全实现了 carbon-relay 和 carbon-aggregator 的功能，性能好很多。

# whisper

whisper 是一个非常简单直观的时间序列存储引擎，整个实现是在一个 800 行的文件。其通过文件系统实现，每个指标是一个独立的文件。如指标 `test.server.cpu-load` 对应于文件 `test/server/cpu-load.wsp`。

为了理解其原理，先有理解 retention policy 的概念，假设以下 graphite 配置文件 `storage-schemas.conf`：

```
[carbon]
pattern = ^carbon\.
retentions = 60s:1d, 1h:7d
```

表明以 `carbon.` 开头的指标，每 1 分钟保存一个点，保存 1 天，1 天之前的数据 1 个小时一个点（意味着 60 个 1 分钟分辨率的点通过 aggregate（聚合方法有 max，sum，avg 等）得到 1 个 1 小时的点）。这个 rentention policy 有两个 archive （其结构为 secondsPerPoint，points），第一次收到指标，会根据 retention policy 创建一个固定大小的文件，以上面为例，该指标总共有 （1d/60s + 7d/1h = 1608）个点，每个点序列化为时间戳和值，大小为 `L + d`，即 12 字节，加上固定的头部大小，即为该文件大小。

每次写，根据指标点的时间戳，找到对应的 archive，根据 secondsPerPoint 归一化指标点对应的时间戳，归一过程为，

```
  timestamp = timestamp - (timestamp % archive['secondsPerPoint'])
```

以这个时间戳，找到该点在该磁盘文件的 offset，写入磁盘文件（如果两个 cabon-cache 会写同一个文件，需要使用文件锁，通过 `WHISPER_LOCK_WRITES` 配置）。时间戳相同的两个点，会写到相同位置，从而相互覆盖。可以通过 `MAX_CREATES_PER_MINUTE`，`MAX_UPDATES_PER_SECOND` 限制写磁盘的速度。当更新的速度超过 `MAX_UPDATES_PER_SECOND` 时，数据点会在 MetricCache 缓存住，用内存换取系统性能。

每个 whisper 文件的存储格式为：

```
File = Header,Data
    Header = Metadata,ArchiveInfo+
		Metadata = aggregationType,maxRetention,xFilesFactor,archiveCount
		ArchiveInfo = Offset,SecondsPerPoint,Points
	Data = Archive+
		Archive = Point+
			Point = timestamp,value
```

以下面的 `storage-schemas.conf` 和 `storage-aggregation.conf` 为例：

```
# storage-schemas.conf
[carbon]
pattern = ^carbon\.
retentions = 60s:1d, 1h:7d

# storage-aggregation.conf
[default_average]
pattern = .*
xFilesFactor = 0.5
aggregationMethod = average
```

以 `carbon.` 开头的指标的存储文件为：

```
Header = Metadata,ArchiveInfo+
    Metadata = average,7d,0.5,2
    ArchiveInfo = headerSize,60,1440
                  offset,3600,168
```

offset 表明该 archive 的第一个数据点在该文件的 offset。由于每个数据点都占固定大小，所以给定时间戳，和 archive，很容易知道该点应该存在什么位置上

# ceres

[ceres](https://github.com/graphite-project/ceres) 是 graphite 官方推出的取代 whisper 的存储引擎,目前处于 experiment 状态，carbon 对 ceres 的维护脚本也还没被合到 master (graphite 现在的开发极其缓慢)，但是，其状态见[这里](https://github.com/graphite-project/ceres/issues/15)。

ceres 依然利用文件系统。所有指标被放在 `CeresTree`，`CeresTree` 包含 `CeresNode`，每个 `CeresNode` 包含多个 `CeresSlice`。其中，`CeresTree`、`CeresNode` 都是目录，一个指标对应一个 `CeresNode`，指标的 `precision`（即 secondsPerPoint） 和 `retention`(即 number of points) 等 metadata 存在 `CeresNode` 目录下的 `.ceres-node` 文件里。目录结构和 whisper 一样，是根据指标名定的，具体的指标点存在 `CeresSlice` 里， `CeresSlice` 的文件名为 `startTime@timeStep.slice`，每个 `CeresSlice` 只存指标点的值（每个点占 8 字节），每个点的时间戳根据 `startTime`，`timeStep` 以及该点在文件中的偏移计算得到。

Ceres 有以下特性：

* 允许一个指标的数据点可以存在不同的文件里。我们知道 whisper 把不同指标存在不同的文件，这也就允许通过一致性哈希，把指标存在不同的 server，而 graphite 集群也是这么做的，在 Ceres，一个指标的 `CeresSlice` 的文件名为 `startTime@timeStep.slice`，这样，相同指标的数据点也可以存在不同文件中，我们可以做一些想象，如把`startTime`、`timeStep` 也加到一致性哈希的 key 里。（虽然这好像增加了复杂性而没多少收益）
* 没有实现 roll-up aggregation 和 data-expiration，。whisper 除了写那个数据点对应的 archive，也会看是否有必要聚合到低分辨率的 archive 里去，从而以低分辨率保存老数据，而 Ceres 写一个数据点时，只是简单的把点存到 `CeresSlice`，由另外的进程负责这些事情。
* 占用的磁盘空间随着指标点的进入逐渐增加，而不是一开始就分配一个指标所需要占的全部空间。
* 比 whisper 少 33% 的空间。由于每个时间点不再存储时间戳，而是通过 startTime + timeStep + 文件偏移，所以存储空间占用会减少，减少为 (时间戳 4 字节 / （4 + 8）= 33%)

Ceres 如何实现 roll-up aggregation 和 data-expiration。代码目前在 [carbon 的 megacarbon 分支](https://github.com/graphite-project/carbon/blob/megacarbon/plugins%2Fmaintenance%2Frollup.py) 。同一指标的不同分辨率的指标点是存在不同 `CeresSlice` 的，rollup 进程读每个 slice 文件，把该文件中已过期的数据点从文件中删除，如果有低分辨率的 slice 文件，则聚合后写入该文件，这样，可以根据配置，过期老数据，或降低老数据的分辨率。
