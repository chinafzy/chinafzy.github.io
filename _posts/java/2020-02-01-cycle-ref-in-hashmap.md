---
title: "HashMap的死锁（JDK7)"
categories: java
tags: Java
---

## 说明
早期JDK的HashMap，在多线程下，会出现各种问题：
+ 数据掉了
+ 数据删了又回来了
+ 死锁

这里分析死锁产生的原理（或者说可能性）之一

## 源码
http://hg.openjdk.java.net/icedtea/jdk7/jdk/file/tip/src/share/classes/java/util/HashMap.java
```java
void transfer(Entry[] newTable, boolean rehash) {
  int newCapacity = newTable.length;
  for (Entry<K,V> e : table) {
    while(null != e) {
      Entry<K,V> next = e.next;
      if (rehash) {
        e.hash = null == e.key ? 0 : hash(e.key);
      }
      int i = indexFor(e.hash, newCapacity);
      e.next = newTable[i];
      newTable[i] = e;
      e = next;
    }
  }
}
```
**我们需要关注的是这个while循环（行4-13）**


## 案例设定
假设有table数组长度是2，有俩个元素，key的hashCode分别是【3】和【7】。  
现在要将其transfer为长度4的数组。

原始的数据结构如下：
```
0
1  	【3】-> 【7】
```

## 程序运行的过程


### while循环第一次运行到最后
```
0
1 	
2
3 	【3】

e  =【7】
```

### 第二个循环结束
```
0
1
2
3 	【7】 -> 【3】

e = null
```
+ e = null，整体循环结束


## 死锁的产生
假设我们有俩个线程，同时运行了transfer操作（这个非常容易发生）。

### 俩个线程都完成第一个循环
数据结构变为
```
0
1 	
2
3 	【3】

e  =【7】
```

### 线程1停留在这里，线程2继续完成后面的工作

线程2里面看见的数据结构是这样
```
0
1
2
3 	【7】 ->【3】
```

根据Java内存模型，节点之间的next指向关系会影响到线程1

因此，线程1里面的数据结构就变成了
```
0
1 	
2
3 	【3】

e  =【7】->【3】
```

### 线程1的二次循环结束
```
0
1 	
2
3 	【7】->【3】

e  =【3】
```
这个时候，e != null，还会继续循环

### 线程1的三次循环结束
```
0
1
2
3 【3】<->【7】

e = null
```
+ e = null，整体循环结束
+ 节点【3】【7】形成相互循环

## 死锁问题爆发
`查询`操作的key位于slot3上时候（例如hashCode是【11】），将产生死循环

## 问题分析
这个问题的根源，在于：
+ 发生问题的key，它们在tranfer到新的数组后还位于一个slot上
+ 链表的头插法
+ 多线程

链表的头插法是其中的关键要素之一，这个不修改，问题就始终存在。下面是jdk7中HashMap的其它版本，这个问题依旧存在。

https://github.com/openjdk-mirror/jdk7u-jdk/blob/master/src/share/classes/java/util/HashMap.java


