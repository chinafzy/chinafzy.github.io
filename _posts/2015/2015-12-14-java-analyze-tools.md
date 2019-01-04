---
title: "非常有用的Java在线故障分析工具"
tags: Java Analyze FAV
categories: java
date:   2015-12-14 00:00:00 +0800
---

在故障分析过程中，积累了一些经验，整理了几个非常有用的工具。

## jtop

如果Java进程占据了大量的CPU时间，而不知道是哪个线程/函数在占据时间，可以使用它来分析java线程的占据CPU线程的前几名。

使用方法：将函数拷贝到一个文件中，然后调用 `source <文件名>`的方式加载函数。

参数说明：
+ java_pid：java进程的pid
+ count：可选项，默认值5，要分析的top线程数的数量。

```bash
jtop(){
  [[ -z "$1" ]] && echo "Usage: $FUNCNAME java_pid [count]" && return 255

  local jid=$1
  local count=${2:-5}

  echo "with count $count"

  tfile=/tmp/"$jid"_$(date "+%Y%m%d_%H%M%S").tdump
  jstack $jid > $tfile
  echo "jstack dump to $tfile"

  for i in $(ps -L -o %cpu,lwp -p $jid | sort -nr | head -$count | awk '{print $1 ";" $2}' ) ; do
    arr=(${i//;/ })
    cpu=${arr[0]}
    lid=${arr[1]}
    lidhex=$(printf "%x" $lid)
    gtext="nid=0x$lidhex"
    echo "==== LWP: $lid($lidhex) === cpu:$cpu ==== "
    sed -n "/$gtext/,/\"/p" $tfile | head -n -2
  done
}

```

案例：

懒得写了，自己试试看。

## jcheck
如果任务处理的慢了（不一定是CPU高），可以用这个函数来“猜测”是否有哪个函数卡住了。

使用方法：将函数拷贝到一个文件中，然后调用 `source <文件名>`的方式加载函数。

参数说明：
+ java_pid：java进程的pid
+ count：可选项，默认值9。抽样抓取的数量，

```bash
jcheck(){
  [[ -z $1 ]] && echo "Usage: $FUNCNAME pid [count] [interval]" and return 255

  local pid=$1
  local count=${2:-9}
  local inter=${3:-0}

  echo "$FUNCNAME $pid $count $inter"

  local bpath=~/tdump/$pid
  mkdir -p $bpath
  for i in $(seq $count)
  do
    echo $i
    jstack $pid > $bpath/$i.tdump
    cat $bpath/$i.tdump \
    | awk '{if ($0 == "") {if(len>20){print "=========="; for(p=0; p <len; p++) print arr[p];}; len=0 } else {arr[len++]=$0;} }' \
    > $bpath/$i.lives
    if (($inter>0)); then sleep $inter; fi
  done

}
```
