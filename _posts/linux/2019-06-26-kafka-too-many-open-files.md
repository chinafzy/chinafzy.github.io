---
title:  Kafka Broker 出现Too many open files
date:   2019-06-26 07:08:11 +0800
categories: linux 
tags: Linux Java
---

## 问题描述

### 环境
+ kafka server 0.10.0.1 （查看libs/kafka_2.10-0.10.0.1.jar）
+ 5个broker
+ 1000个partition，3份的存储
+ 部分partition每秒数据有3000，一半以上的大概1000条
+ 数据每条大概是100B
+ 200个partition的数据被实时消费
+ 使用机械硬盘

### 问题表现
+ 外部无法连接
+ 此时第一个kafka broker上的日志`server.log`大量出现错误
```
[2019-06-21 18:03:29,341] ERROR Error while accepting connection (kafka.network.Acceptor)
java.io.IOException: Too many open files
        at sun.nio.ch.ServerSocketChannelImpl.accept0(Native Method)
        at sun.nio.ch.ServerSocketChannelImpl.accept(ServerSocketChannelImpl.java:422)
        at sun.nio.ch.ServerSocketChannelImpl.accept(ServerSocketChannelImpl.java:250)
        at kafka.network.Acceptor.accept(SocketServer.scala:323)
        at kafka.network.Acceptor.run(SocketServer.scala:268)
        at java.lang.Thread.run(Thread.java:748)
```
+ 查看kafka server的文件句柄`lsof -p <pid>`，里面有近十万的`can't identify protocol`
+ 消费程序无法获取数据
+ 到一定时候，有问题的broker可能崩溃。
+ 重启出问题的broker后，问题解决。

## 排查过程

因为数据是多副本，所以考虑将问题节点下线，但是下线后遇到问题，消费程序都不能从kafka消费。

使用`jstack`连续多次抓取消费者Java进程，发现，

但是这时用`kafka-console-consumer.sh`来消费是可以。

继续排查后，发现不同的消费方式会有不同的结果（在后续版本中，这些参数混乱被纠正了）：

+ 使用参数`--zookeeper localhost:2181`可以消费
+ 使用参数`--bootstrap-server localhost9092 --new-consumer`的方式不可以消费

有伙伴怀疑，`__consumer_offset`是单点的，使用`kafka-topic.sh --describe`看了之后，发现的确如此，而且单点位置放置在第一个broker上，因此当第一个broker出问题后，整个kafka集群不能消费。

使用`kafka-reassign-partitions.sh`将`__consumer_offset`做三份，打散到多个broker后。问题解决。

## 深入分析

### `__consumer_offset`单点导致的网络连接问题





### `__consumer_offset`单点问题的产生

