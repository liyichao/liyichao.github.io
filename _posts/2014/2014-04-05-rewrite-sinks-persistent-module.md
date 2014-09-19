---
layout: post
title: "rewrite sink's persistent module"
category:
tags: []
---
{% include JB/setup %}

sink也是个消息订阅与发送队列，与kids的区别就是每条消息，在发送给订阅者前，都得先存盘。原先的存储模块是用`Kyoto Cabinet`。这是一个key-value数据库，底层由hash表或B+树实现。sink的消息有先后顺序的，这个顺序与消息的产生时间一致。sink支持的一个功能叫session。客户可以make session，sink会传给客户一个id号，这个id号其实就是cursor，表示在消息队列里的一个位置，当客户以后load这个session，那么自cursor以后的消息，都会被推送到客户。为了实现这个先后顺序，sink使用时间作key，建了两个数据库，一个存topic，一个存data。

文档说它runs very fast。可是不管多fast，还是受限于底层hash或B+的实现。对于sink来说，并不需要根据key去随机访问value。就说session功能，也是自某个时间点开始，顺序的访问消息队列。

这种场合，直接用一个append-only-file够了。消息格式是：

>message_size SEP topic_size SEP topic SEP data\n

我是直接用write成文本的，存成二进制可以用`fread`和`fwrite`，用fwrite的话直接打开存储文件就看不见message_size，不好调试。没用标准IO确是一个失误。