---
title: "Redis Server的连接不能释放(connection leak)"
categories: redis
tags: Redis
---
**这个问题在kafka server的早期版本(0.9)中也存在 [Enable keepalive socket option for broker to prevent socket leak](https://issues.apache.org/jira/browse/KAFKA-2096){:target="_blank"}**

## 问题描述
线上系统发现Redis无法连接，查看`info clients`后发现Redis Server上的连接数已经上万。  

而且特别奇怪的是：  
从Redis Server上看见的网络连接 `netstat -anop | grep :6379`，在对方服务器那边看，却没有连接。

**有上万个幽灵连接，在Redis Server上和它所在的主机上**。

## 紧急处理
直接将Redis Server的max_clients设置为更大的数，可以接受更多的连接。问题缓解。

## 原因分析
网络长时间连接中，有可能是断开了，但是连接的俩端宿主不知道，就会形成幽灵连接。Redis Server上的连接如果成了幽灵连接，就会占据他的连接名额。

Linux针对这个，有一个[keepalive方案](http://tldp.org/HOWTO/TCP-Keepalive-HOWTO/usingkeepalive.html){:target="_blank"}：
+ 在Idle到一定时间后(tcp_keepalive_time，默认7200秒)，主机给连接发一个测试包
  + 如果测试包得到了回复，Idle计时器和测试计数器都置零，测试重新开始
  + 如果没有得到回复，测试计数器+1
    + 计数器加到一定程度阀值（tcp_keepalive_intvl，默认9次），认定连接失效，通知系统
+ 间隔一定时间后(tcp_keepalive_intvl，默认75秒)，再次重复上一步的测试。
+ 在这个过程中，如果有一次正常的网络请求能成功，等同于测试通过。

从描述可以看出来，keepalive的作用是俩个：
+ 测试连接是否正常
+ 给对方发`心跳包`，保持连接。

操作系统内置了keepalive功能，需要运行在上面的程序来调用这个功能。

查看Redis Server的keepalive的方法如下： 
```bash
127.0.0.1:6379> CONFIG GET tcp-k*
1) "tcp-keepalive"
2) "180"
```
（Redis Server从版本3.2.1开始，默认开启对keepalive的调用。）

keepalive功能开启后，redis server的幽灵连接问题就得到了解决。  
配置参数一般不要低于60

## 原有连接的释放
设置了keepalive后，Redis Server上的新建连接会使用keepalive特性，但是已经产生的连接依然是没有keelalive特性——包括已经死了的连接。这些虚假的连接对于Redis Server的性能影响微乎其微，不过有办法将其去除。
  
原理是利用Redis Server的`timeout`机制:  
如果是已经死了的连接，肯定长时间没有工作了。

1. 查看连接数 
```bash
127.0.0.1:6379> info clients
# Clients
connected_clients:1000
client_longest_output_list:0
client_biggest_input_buf:0
blocked_clients:0
```
当前有1000个连接，`connected_clients:1000`

2. 设置`timeout`参数
```bash
127.0.0.1:6379> CONFIG SET timeout 86400
OK
```
这里设置为86400，也就是【一天之内，如果连接没有数据处理，则被认为timeout，清理】。

3. 重新检查连接数  
如果发现连接数减少，表示已经有部分`不活跃`/`死亡`的连接已经被清理了。  
同时可以检查Linux服务器的连接数，也有对应的连接被清理了。
```bash
$ netstat -anop | grep 6379 
```

4. 将`timeout`设置为一个更小的数，再做试验。  
一般可以设置到2小时，再小可能就出问题，误杀。

**注意：一定不要直接设置一个比较小的值，否则可能会出现很大麻烦。**

设置timeout，可能会导致正常状态的客户端连接在长时间不工作后被Redis Server杀了。对应的处理办法是在客户端那边设置idleTest，例如JedisPool#setTestWhileIdle()。  
当然更好的办法是不要在Redis Server上设置timeout。


## 附录

### 怎么样查看一个网络连接是否是keepalive的
使用命令`netstat -o`可以看到网络连接的keepalive状态。

+ 将keepalive设置为1800，然后对Redis Server建立一个连接
```bash
$ netstat -o
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       Timer
tcp        0      0 localhost:6379          localhost:35548         ESTABLISHED keepalive (254.34/0/0)
```

+ 断开连接，将keepalive设置为0，然后对Redis Server建立一个连接
```bash
$ netstat -o
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       Timer
tcp        0      0 eb30bd5c7319:6379       172.17.0.4:48078        ESTABLISHED off (0.00/0/0)
```

在Timer列上，有`keepalive`的是一个keepalive连接，`off`的是一个非keepalive连接。