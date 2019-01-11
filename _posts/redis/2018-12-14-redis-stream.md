---
title: "redis stream"
categories: redis
tags: Redis
---

## 从一个需求案例说起

**假定我们有个需求，一个系统下有这些东西：**
+ 一个分布式任务模块，下面有多个执行节点，每个任务只能被某个节点处理。  
—— 很明显，redis的List适合做这个，List的数据只能被POP一次。
+ 一个config配置模块，如果配置信息发生改变，需要每个节点都知道这个**事件**，然后大家刷新数据。  
—— 很自然，Redis的Pub/Sub适合做这个，消息会同时发给全部的监听节点。

很快就做好了，跑的不错。

**然后我们有了个新的需求：**
+ 需要监控这些指令和事件。客户会追问，每个指令什么时候来的，什么时候被处理。  
—— 对应的解决办法就是增加日志，然后检查日志。好消息是：日志只要生成一份就可以了，可以给好几个场景检查。  

但是为什么Redis的数据本身不能像日志文件那样的可以重复消费呢？  
有人会说，kafka可以支持这个。  
Redis也说：5.0里的Steams也支持这个了，就是完全参考了Kafka的。

Kafka的设计思路特别简单神奇，它的数据结构本质上是一个非常长的数组，
+ 这个数组的编辑方式只有从后面追加，和从前面截断放弃  
——连续读写发挥了机械磁盘的最好性能。
+ 读取方式是，消费者保存自己的数组位置，
  + 一组消费者共享一个位置，就可以实现组内不重复消费；
  + 不同组的位置相互不影响，消费行为是完全隔离的，也就实现了重复消费。

我们对Redis的三个数据类型进行对比：

 类型 | 多人重复消费 | 可以保留
---|---|--
List | 否，每条数据只能被某一个消费者获得 | 是，未被消费的数据一直保留
Pub/Sub | 是，数据会发给每个消费者 | 否，消息发出的瞬间就没有了
Stream | 都支持 <br />一个group中的消费者得到的数据是不同的；<br/>不同group的消费相互隔离 | 是，而且数据被消费后依然存在

这个文档就是针对Stream的使用（主要是数据读写）。

**更进一步的概念，有兴趣的可以去找TimeSeriesDB(TSDB)看看**。

## 怎么样看文档
看文档的时候，最好能一边看一边实践，这里准备了一些脚本：

+ 批量加入stream数据

这个命令向名为`s1`的stream中插入100个 `f1=v1` 这样的数据
```lua
EVAL 'for i = 1, ARGV[1] do redis.call("xadd", KEYS[1], "*", "f1", "v1") ; end' 1 s1 100
```
可以参数设定插入的stream名称和数据条数

+ 批量加入stream数据——简化版

这个命令向s1中添加3000个数据。
```lua
EVAL 'for i = 1, 3000 do redis.call("xadd", "s1", "*", "f1", "v1") ; end' 0
```

## 使用案例


## API

### 编辑

#### XADD

```sh
XADD key ID field string [field string ...]
```
+ 说明：向stream中插入一条数据，其中可以包含多组field-value值对
+ 参数
  + key：stream的名称
  + ID：插入数据的ID，一般建议为*，由系统自动生成
  + field + string：数值对，可以有多组
+ 返回值：string，插入的数据ID

```
redis> XADD mystream * name Sara surname OConnor
"1542031898153-0"
redis> XADD mystream * field1 value1 field2 value2 field3 value3
"1542031898155-0"
redis> XLEN mystream
(integer) 2
redis> XRANGE mystream - +
1) 1) "1542031898153-0"
   2) 1) "name"
      2) "Sara"
      3) "surname"
      4) "OConnor"
2) 1) "1542031898155-0"
   2) 1) "field1"
      2) "value1"
      3) "field2"
      4) "value2"
      5) "field3"
      6) "value3"
```

#### XDEL(不重要)
```
XDEL key ID [ID ...]
```
+ 说明：从stream中删除指定ID的条目
+ 参数
  + key
  + ID 要删除的entry的ID
+ 返回：int，删除的数据条数

#### XTRIM 
```
XTRIM key MAXLEN [~] count
```
+ 说明：给stream限定一个容量，超出这个容量范围的 *老* 数据会被删除
+ 参数 
  + key
  + MAXLEN，关键字
  + [~]，关键字，可选项，可以理解为*约等于*，使用这个关键字后， trim后的结果不是**严格的**等于指定的数量，而是允许超出几十。**官方强烈建议使用**，因为stream内部数据结构的特性，这样的性能比较好。  
The ~ argument between the MAXLEN option and the actual count means that the user is not really requesting that the stream length is exactly 1000 items, but instead it could be a few tens of entries more, but never less than 1000 items. When this option modifier is used, the trimming is performed only when Redis is able to remove a whole macro node. This makes it much more efficient, and it is usually what you want.
  + 限制的数量
+ 返回值：删除的条目数量

```
127.0.0.1:6379> EVAL 'for i = 1, 3000  do redis.call("xadd", "s1", "*", "f1", "v1") ; end' 0
(nil)
127.0.0.1:6379> XLEN s1
(integer) 3000
127.0.0.1:6379> XTRIM s1 maxlen ~ 1000
(integer) 1919
127.0.0.1:6379> XLEN s1
(integer) 1081
127.0.0.1:6379> XTRIM s1 maxlen ~ 1000
(integer) 0
127.0.0.1:6379> XTRIM s1 maxlen ~ 1000
(integer) 0
127.0.0.1:6379> XLEN s1
(integer) 1081
127.0.0.1:6379> XTRIM s1 maxlen 1000
(integer) 81
127.0.0.1:6379> XLEN s1
(integer) 1000
```

### 读取

#### XRANGE和XREVRANGE
```
XRANGE key start end [COUNT count]
XREVRANGE key start end [COUNT count]
```
ID排序，从大到小或从小到大获取一个范围内的记录。


+ 说明：在stream中获取一个范围值
+ 参数：
  + key
  + start和end：开始和结束的ID，
    + 可以对start使用-，表示最小ID，0-0
    + 可以对end使用+，表示最大ID，18446744073709551615-18446744073709551615
  + [COUNT count] 可选参数，指定返回的数据数量不超过这个限度


## 待续