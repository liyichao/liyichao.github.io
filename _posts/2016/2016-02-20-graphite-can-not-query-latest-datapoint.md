---
layout: post
title: "graphite can not query latest datapoint"
category: 
tags: []
---
{% include JB/setup %}

说一个遇到的 graphite 图表滞后的问题。

我们有用 [graphite](https://graphite.readthedocs.org/en/latest/) 做指标存储，用 [carbon-c-relay](https://github.com/grobian/carbon-c-relay) 负载均衡到多个 carbon-cache 实例，把整个系统的瓶颈转换到磁盘 IO 上。relay 配置如下：

```
cluster graphite
    carbon_ch
			graphite-host:2103=1
			graphite-host:2203=2
			graphite-host:2303=3
			graphite-host:2403=4
			graphite-host:2503=5
			graphite-host:2603=6
    ;

match * send to graphite;
```

relay 部署在它独有的机器，graphite-web 和 carbon-cache 部署在 `graphite-host` 上，carbon-cache 端口配置为：

```
[cache:1]
LINE_RECEIVER_PORT = 2103
CACHE_QUERY_PORT = 7102

[cache:2]
LINE_RECEIVER_PORT = 2203
CACHE_QUERY_PORT = 7202

[cache:3]
LINE_RECEIVER_PORT = 2303
CACHE_QUERY_PORT = 7302

[cache:4]
LINE_RECEIVER_PORT = 2403
CACHE_QUERY_PORT = 7402

[cache:5]
LINE_RECEIVER_PORT = 2503
CACHE_QUERY_PORT = 7502

[cache:6]
LINE_RECEIVER_PORT = 2603
CACHE_QUERY_PORT = 7602
```

graphite-web local_settings.py 相关配置为：

```
CARBONLINK_HOSTS = ["127.0.0.1:7102:1", "127.0.0.1:7202:2", "127.0.0.1:7302:3", "127.0.0.1:7402:4", "127.0.0.1:7502:5", "127.0.0.1:7602:6"]
```

graphite-web 和 carbon-cache 其他配置不是本文关注的目标，具体见 [graphite cluster 搭建](https://grey-boundary.io/the-architecture-of-clustering-graphite/)。
指标分辨率是默认 10s 一个点，最近发现我们的的图表老是滞后 2 分钟左右，比如当前是 16:42:00，最新只能看到 16:40 的点，理论上能看到 16:41:50 的点。

根据 graphite 的原理，graphite-web 不仅会查 whisper，还会和 carbon-cache 通信，所以，只要 carbon-cache 收到了点，即使还没来得及写到文件，也是可以查到的。首先确定 carbon-cache 收到的点是不是滞后，通过 tcpdump：

```
sudo tcpdump -i any port 2103 or port 2203 or port 2303 or port 2403 or port 2503 or port 2603 -A  | grep test.test
```

其中假设 carbon-cache 有 6 个实例，`test.test` 是测试指标，一开始调查时是先从发现问题的图表的指标开始的。得到时间戳，通过 irb 转换成易读的格式：

```
➜  blog git:(master) ✗ irb
irb(main):001:0> Time.at 1455949862
=> 2016-02-20 14:31:02 +0800
```

发现收到的点是最新的点，从这时就开始怀疑是 graphite-web 没从 carbon-cache 查到点，可能是 carbon-c-relay 发送到的 carbon-cache 实例，和 graphite-web 去查的那个实例不一致。这时就想知道某个指标到底发往哪个实例了，根据印象，知道 graphite 有个集群工具 [carbonate](https://github.com/jssjr/carbonate)，发现果然有个工具完成这个功能：`carbon-lookup`。用 virtualenv 生成一个 .venv，用 pip install 安装完，运行 carbon-lookup 报找不到 carbon 这个模块，反复烦恼一番最后发现 carbon 安装在了 `.venv/lib/python2.7/site-packages/opt/graphite/lib/`，于是在 `.venv/bin/activate` 设置下 PYTHONPATH 解决该问题。

于是生成以下配置去查 `test.test`:

```
[main]
DESTINATIONS = 127.0.0.1:7102:1, 127.0.0.1:7202:2, 127.0.0.1:7302:3, 127.0.0.1:7402:4, 127.0.0.1:7502:5, 127.0.0.1:7602:6
REPLICATION_FACTOR = 1
```

```
$ carbon-lookup test.test
127.0.0.1:7302:3
```

然后在另一个端口起一个测试用的 carbon-c-relay，测试 carbon-c-relay 发往哪了：

首先准备好抓包：

```
sudo tcpdump -i any port 2103 or port 2203 or port 2303 or port 2403 or port 2503 or port 2603 -A  | grep -C 100 test.test
```

这里 grep 抓上下文是为了知道是在哪个端口收到了这个指标，从而知道是哪个实例收到了该指标。

起测试用的 carbon-c-relay:

```
./relay -f test_relay.conf -d
```

手动发送指标：

```
echo "test.test 1 `date +'%s'`" |nc -q0 localhost 2003
```

抓包发现是第二个实例收到包，这时可以肯定 carbon-c-relay 发送错实例了，但当时还有点犹豫，于是用 graphite 带的 relay 做了下实验，确定了这一点。

这时就想不通了，但由于之前已经看过 graphite 源码，所以并不太慌张，先看 carbonate 的[代码](https://github.com/jssjr/carbonate/blob/master/carbonate%2Fcluster.py#L16)，关键函数有两个carbon 的 `ConsistentHashRing` 的构建，和 `ConsistentHashingRouter.getDestinations`。由于之前 relay 跑 debug 时，会打出 hash ring，于是想到把 lookup 里的 hash ring 打出来。

```
[   (18, ('127.0.0.1', '5')),
    (239, ('127.0.0.1', '1')),
    (551, ('127.0.0.1', '1')),
...
]
```

而 carbon-c-relay 是：

```
104@graphite-host:2303=3      151@graphite-host:2403=4 
```

按源码的方法，计算了下 `test.test` 的 hash key，是 53260，按照 hash ring，carbon-c-relay 是在实例 2，而 lookup 应该是在实例 3，因此，是 hash ring 生成的问题。graphite hash ring 相关代码是[这里](https://github.com/graphite-project/carbon/blob/master/lib%2Fcarbon%2Fhashing.py#L24)，carbon-c-relay 是 [carbon-c-relay hash ring 生成相关的代码](https://github.com/grobian/carbon-c-relay/blob/master/consistent-hash.c#L227)，琢磨了一段时间，知道了 18 是 `('127.0.0.1', '5'):i` 的 hash （i 从 0 到 99），如果 hash 算法一样，前面的数字不一样，那一定是后面的字符串不一样，于是逐个字对比，就在那一瞬间，知道是 127.0.0.1 那一段不一样，因为部署在不同机器上，carbon-c-relay 不可能写 127.0.0.1，写的是 `graphite-host`。

于是，修改 graphite-web local_settings.py 配置：

```
CARBONLINK_HOSTS = ["graphite01:7102:1", "graphite01:7202:2", "graphite01:7302:3", "graphite01:7402:4", "graphite01:7502:5", "graphite01:7602:6"
```

上线后，再看图表，恢复正常！