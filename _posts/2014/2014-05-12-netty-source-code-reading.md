---
layout: post
title: "netty 4 源码阅读"
category:
tags: []
---
{% include JB/setup %}

# Overview

Notes: netty 4.0.19.Final 版本 

先来看下官网的标语：

>Netty is a NIO client server framework which enables quick and easy development of network applications such as protocol servers and clients. It greatly simplifies and streamlines network programming such as TCP and UDP socket server.

我们来理解下。看过 [kids](https://github.com/zhihu/kids) 或 redis、或 nginx 代码的人就知道，要写一个网络服务器，特别是 TCP server，是很繁琐的。需要实现线程模型，需要实现自己的协议，我们多想，像写 HTTP server 一样，写一个 TCP server 时，只需要写一个 handler，一切就完成了，Netty 就是这么一个让编写 TCP Server 更简单的一个网络框架。

这篇文章会从整体上对 Netty 进行讲解，而不会涉及，例如 Zero Copy Buffer 是怎么实现的。

# 基本概念

理解一个系统，先得理解它的基本概念。

`EventLoopGroup`：具有事件循环的 executor。一个网络服务器总是要起线程或进程去处理请求，这个 `EventLoopGroup` 就是这么一个运行的实体，类似 `EventExecutorGroup`，只是它可以注册 `Channel`，即，提供了事件循环，当 `Channel` 可读或可写时会执行回调。

`Channel`：套接字的封装。`Channel` 关联了一个 `ChannelPipeline`，即当`Channel` 可读或可写时，会触发 `ChannelPipeline` 里各 `ChannelHandler` 的调用，简单来说 `ChannelPipeline` 就是 `ChannelHandler` 的链表，当 `Channel` 状态变化时，会依次调用这个链表里各 `ChannelHandler` 的对应方法。这个 `ChannelPipeline` 就是 Netty 建立的一个 pipeline 机制，所有业务逻辑，封装在 `ChannelHandler`，并加入到 `ChannelPipeline` 里，从而一个 TCP 服务器就建立起来了。

`Future`：提供了异步编程的机制。像 Redis 那样，直接操作 ioloop，注册套接字事件，是很底层的，Future，就相当于是对异步执行的一个封装。`Future` 有 `success`，`failure`，未完成等状态，`Future` 可能在另一个线程被 `setSuccess`，在本线程可以 `await` `Future` 直到它完成。一般来说编程模式是这样的：在本线程调用一个函数，该函数是异步的，因此返回一个 `Future`，那个函数会把 `Future` 放到某个工作线程，或 `EventLoop` 里去执行，当然，本线程一般不会等待 `Future`，因为这样就不是异步了，会卡住当前线程，本线程一般会注册一个回调，当 `Future` 完成时该回调会调用，netty 支持该回调在 `EventLoop` 线程或者在工作线程执行。

如果看过 [Fingale](http://twitter.github.io/finagle/guide/Futures.html)，那么应该会对它的 `Future` 印象深刻，人家的 Future 可是支持 `Map` 等操作的，真是令人咋舌。


# 其他

理解了基本概念，感觉要讲的其他东西就基本没有了。比如，就像 [heka](https://github.com/mozilla-services/heka) 一样，Netty 会把 `ChannelInboundHandler` 的第一步叫做 `Decoder`，即，由字节转换为 message，其实就是协议解析了。
