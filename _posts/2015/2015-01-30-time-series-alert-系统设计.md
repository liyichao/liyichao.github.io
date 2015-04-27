---
layout: post
title: "Time Series Alert 系统设计"
category: 
tags: []
---
{% include JB/setup %}

# 开源实现

## Cabot

[Cabot](https://github.com/arachnys/cabot)。代码很混乱，Web 是用 Django 写的，第三方依赖多，用了一下午时间才把它跑起来。

* 一个 Services 有多个指标，人和 Services关联，一个指标可以属于多个 Services，Instance 表明 Service 的实例数。概念只有 Services, Checks，Instance，极其简洁。
* 可以用 graphite 函数指定指标
* 用 celery 任务队列实现，foreman 进程管理

## Bosun

[Bosun](https://github.com/bosun-monitor/bosun)。很喜欢。用 Docker 跑的，5 分钟就跑起来了（Docker 我爱你）。

## Alerta

[Alerta](https://github.com/guardian/alerta)。


# 设计

