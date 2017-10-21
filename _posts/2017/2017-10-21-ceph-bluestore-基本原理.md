---
layout: post
title: "ceph bluestore 基本原理"
category:
tags: []
---
{% include JB/setup %}

# 磁盘里都放什么

BlueStore 用了三个分区。

* DB：首 BDEV_LABEL_BLOCK_SIZE 字节存 label，接着 4096 字节存 bluefs superblock，superblock 里会存 bluefs journal 的 inode，inode 指向 WAL 分区的物理块，DB 之后的空间归 bluefs 管，其中 db 目录存  meta 信息（通过 rocksdb），包括 Block 分区的 freelist。OSD 启动后，bluefs 会读 journal 的 inode，里面包含了文件内容所包含的物理块，bluefs 会通过设备文件直接读这些物理块，journal 里是 bluefs_transaction_t 列表，bluefs 从第一项一直执行到最后一项，bluefs 就在内存里建立了其所有元数据。它的元数据包含哪些目录，每个目录有哪些文件，每个文件的内容在哪些物理块。bluefs 目前是完全为 rocksdb 服务的，通过 [rocksdb environment](https://github.com/facebook/rocksdb/wiki/basic-operations#environment) 为 rocksdb 提供了基于物理块设备的文件操作。

* WAL：首 BDEV_LABEL_BLOCK_SIZE 字节存 label，其余都是 Bluefs 管，inode 1 存的是 bluefs journal 文件， db.wal 目录存 RocksDB 的 WAL。

* Block： 首 BDEV_LABEL_BLOCK_SIZE 字节存 label，从中间开始的size * (bluestore_bluefs_min_ratio + bluestore_bluefs_gift_ratio) 的空间归 bluefs 管，由 bluefs_extents 表示，其余存 Data，其空闲块列表存 RocksDB，由 Freelistmanager 管理。

由上可见，OSD 启动时，BlueStore 的初始化过程是读 DB superblock，然后 reply bluefs journal，DB 分区，WAL 分区以及 Block 分区中 bluefs 的内容都可以访问了，于是基于bluefs db，db.wal 目录的 RocksDB 就可以初始化了，RocksDB 初始化后，对象的元数据载入了，Block 分区的空闲块列表也载入了，Bluestore 就可以开始对外服务了。

# BlueStore 如何对外服务

基于磁盘存放的内容，我们可以猜想如何往 bluestore 写数据。Bluestore 对外提供的是k-v 接口。通过 RocksDB 里对象的元数据，找到或分配 key 对应的对象的物理块地址，先把对象内容写物理块，然后用 RocksDB 的 Transaction 把 journal 和其他元数据原子性写入磁盘。这里需要先写物理块是因为如果先写 RocksDB，由于 journal 里不会存对象的实际内容，只会存元数据，所以如果写 RocksDB 后进程挂了，那么实际上对象的数据并没有写入，但是 bluestore 已经认为对象修改成功了，导致对象旧数据丢失了，新数据未写入成功。而如果写物理块失败了，或者写成功了而 RocksDB 失败了，那么只需要保证新写的物理块不覆盖原来数据的物理块，因为对象引用的还是旧物理块，只需直到 RocksDB 的 Transaction 成功才返回成功，否则一直重试，这样就能保证写对象数据和元数据的原子性。

# 友情链接

* [ceph 存储引擎 bluestore 解析](http://www.sysnote.org/2016/08/19/ceph-bluestore/)
* [Bluestore: a new storage backend for ceph](https://www.slideshare.net/sageweil1/bluestore-a-new-storage-backend-for-ceph-one-year-in)
