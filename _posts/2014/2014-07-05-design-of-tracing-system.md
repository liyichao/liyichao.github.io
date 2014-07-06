---
layout: post
title: "design of tracing system"
category: 
tags: []
---
{% include JB/setup %}

####缩写

* CAT：[大众点评网监控平台](http://www.infoq.com/cn/presentations/public-comments-monitoring-platform-analyse)

####目标

* 追踪RPC调用，得到依赖关系和运行时间栈，类似twitter zipkin。运行时间栈：假设A调用B和C，B调用D。运行时间栈就是指A花费多少时间，其中B,C分别花了多少时间，B花的时间中，D花了多少时间。
* 具有关闭trace的功能

####消息格式

* trace id：一个trace id代表一次RPC调用，后续的RPC调用用同一个trace id。
* span id
* type：服务名:方法名
* message

####消息如何送出去

CAT把消息放到异步队列。

可以把消息发到Kids agent。


####消息如何集中和处理

可以在kids server端直接处理。

怎么处理：

可以用一个字典，把相同trace id的消息放到一个桶里，称为一个Trace对象。对于一个Trace对象，用一个数组表示，span id 从0开始生成，放到数组对应位置，不够就realloc。同一个span id可能有多个元素，用链表链起来。

从Trace对象就可以得到依赖关系和运行时间栈了。Trace对象存在mysql里。

####如何埋点

在RPC框架里埋点。

####如何生成全局唯一trace id以及如何传输trace id
trace id生成：貌似以前sink有生成unique id的实现。

在RPC调用的时候还需要传输以下内容：

* trace id
* span id。这样新的span id就是father_span_id + 1

所以也许需要改wish传输协议。











