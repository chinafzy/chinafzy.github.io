---
title:  "利用AWK处理日志"
date:   2020-05-06 16:27:11 +0800
categories: linux tips awk
tags: Linux Tips
---

## 前言

大数据的功能自然比awk强大很多，但是它的工作流程过于漫长，awk的快捷灵活在部分情况下更合适：
+ 线上的紧急故障排查：这个时候肯定来不及在大数据里面慢慢折腾
+ 新业务分析：有时候新业务的数据不一定来得及进大数据

学好awk，对于运维和高级后端开发来说都非常必要。

本文档结合实际工作里面的案例，讲述如何使用AWK来进行日志做`性能分析/线上排障`和`业务分析`。

我们线上的nginx（见附录）是以` " |# " `来分隔字段的，主要字段有：
```conf
log_format  main 
  '$remote_addr |# $remote_user |# $year-$month-$day $hour:$minute:$second |# $request |# '
  '$status |# $request_time |# $body_bytes_sent |# $http_referer |# $http_user_agent |# $http_x_forwarded_for |# '
  '$http_host |# $upstream_addr |# $upstream_status |# $upstream_response_time |# $scheme |# $content_type';
```

+ $remote_addr：来源IP
+ $http_x_forwarded_for：如果请求是经过了反向代理，这个字段解析原始的来源IP
+ $year-$month-$day $hour:$minute:$second：访问时间
+ $request：请求的内容，URI、方法(POST/GET...)、协议版本
+ $status：返回的状态码
+ $request_time：【重要】请求的全部处理时间（包括了网络传输）
+ $body_bytes_sent：返回的响应体大小
+ $upstream_response_time：【重要】nginx后端服务器的响应时间
+ $upstream_status：nginx后端服务的响应状态码
+ $upstream_addr：nginx后端服务器的地址端口

这个日志也是非常有实用价值的。

## 【业务分析】漏斗分析
开席先来个硬菜。

`漏斗分析`是新业务分析的非常有用的一个手段，尤其是对于页面交互设计来说，可以看到我们的每个步骤的`流失/留存/转换`。  
比如我们做了一个拉新活动，有3个页面：起始页 -> 抽奖页 -> 提交简单注册信息，我们会关注用户在哪一步流失了。

`funnel.awk`会针对用户访问多个location的情况进行过滤，分析出来每一步的`留存率`
```awk
# awk -f funnel.awk -v v_locs="loc1;loc2;loc3" nginx1.log nginx2.log ...

function f_locIdx(loc, _i) {
  for (_i = 1; _i <= lsize; _i++) {
    if (loc ~ locs[_i]) return _i;
  }

  return 0
}

# 获取一组用户行为的traceid，这里使用$request里面的udid参数来充当
function f_getTid(_arr1, _arr2) {
  split($4, _arr1, "udid=")
  if (!_arr1[2]) {
    printf "bad request without udid: %s \n", $4 >> log_file
    return;
  }

  split(_arr1[2], _arr2, "&")

  return _arr2[1]
}

BEGIN {
  FS = " [|]# "

  if (!v_locs) {
    print "需要传入参数 v_locs " > "/dev/stderr"
    exit 111
  }
  split(v_locs, locs, "[;,]")
  lsize = length(locs)

  # 使用 -v v_log_file=logfile 将调试信息打印到文件中
  #  此参数设置为 off 将关闭打印
  #  默认将打印到 /dev/stderr 中，
  log_file = !v_log_file \
    ? "/dev/stderr" \
    : v_log_file == "off" \ 
      ? "/dev/null" \
      : v_log_file
}

{
  locIdx = f_locIdx($4)
  if (locIdx < 1) next;

  tid = f_getTid()
  if (!tid) next;

  # 记录这个tid+location的访问
  rs[tid, locIdx]++
  if (locIdx == 1) {
    # 记录第一步访问的tid
    step1[tid]
  }
}

END {
  for (tid in step1) {
    for (i = 1; i <= lsize; i++) {
      if (!rs[tid, i]) continue;

      counts[i]++;
    }
  }

  for (i = 1; i <= lsize; i++) {
    printf "%10s %s \n", sprintf("%'d", counts[i]), i == 1 ? "" : ((counts[i] * 100 / counts[i - 1]) "%")
  }
}
```
这个脚本的原理是：
+ 首先为一组行为指定一个`trace_id`（简称`tid`），作为行为跟踪。范例里面用udid来模拟，不是很严谨。
+ 记录每个tid在location下的访问记录
+ 记录下来，每个曾经访问过step1的tid
+ 最后根据step1的访问记录，逐步计算后续步骤的`留存`

输出结果是
+ 第一列：每个步骤的发生次数，基于`trace_id`的`unique`统计
+ 第二列：`留存率`百分百

使用范例：
```bash
cat access.log | \
  head -1000000 | \
  awk \
    -f funnel.awk \
    -v v_locs="appOpen.action;updateToken.action;encryptedHomePage.action"
```


## 切割access.log文件
如果日志文件比较大，将其按照小时或更小的时间单位来切割。  
某些老旧的日志系统，改造比较麻烦，就可以用这样的方式来处理。  

`split-by-hour.awk`
```awk
# awk -f split-by-hour.awk access.log
# 用字符串 " |# " 来切割字段
BEGIN { FS = " [|]# "}

{
  # 切割日期时间，获取hour_minute_second
  split($3, datetime, " ")
  split(datatime[2], hms, ":")

  print $0 >> hms[1]".log"
}
```

**实用技巧：在AWK文件的开始加上注释，说明文件的使用方式。比较复杂的业务，还要加上备注说明。**


## 摘取部分日志
假设我们摘取早高峰（7点8点的日志）

`select-timerange.awk`

```awk
# awk -f select-timerange.awk access.log
BEGIN { FS = " [|]# "}

{
  # 切割日期时间，获取hour_minute_second
  split($3, datetime, " ")
  split(datatime[2], hms, ":")

  if (hms[1] > 8) exit

  if (hms[1] < 7) next

  print $0;
}
```
**实用技巧：AWK永远要优先考虑`退出处理 exit`。**  
对于有几千万数据的日志来说，尽早完成工作，退出处理，是必须的。

**实用技巧：`exit`可以带一个参数，作为AWK运行后的shell返回值。**
```bash
$ awk 'BEGIN { exit 111 }'; echo $?
111
```


## 【线上故障】跟踪响应慢的请求

**对于线上系统来说：慢就是错。**  
没有任何理由，响应慢，用户感受就差，扯什么都没用。

因此，跟踪慢请求，是最基本的一个工作。

一般我们是用这个命令来做：`tail -f access.log | awk -f select-slow.awk | tee slow.log`
+ tail -f access.log，将实时跟进线上的日志
+ awk -f select-slow.awk，将过滤选择比较大的请求
+ tee slow.log，将把输入的内容同时发送到控制台和文件。    
  这个是一个非常有实用价值的命令。与此类似的方式是开俩个命令，一个直接输入到文件，另外一个`tail -f`这个文件。

```awk
# awk -f select select-slow.awk access.log
# awk -f select -v v_second=0.3 select-slow.awk access.log

BEGIN {
  FS = " [|]# "
  # 使用参数v_ms来指定最小的时间，默认值是0.1，也就是100毫秒
  second = v_second ? v_second : 0.1
  printf "选择响应时间 >= %s 的请求\n\n", ms > "/dev/stderr"
}

($6 >= second) {
  print $0
}
```
上面这个awk将选择响应时间在一定值（默认0.1秒）之上的请求。

**实用技巧：AWK可以通过传入参数的对部分关键值进行灵活控制。**

**实用技巧：AWK可以往 `/dev/stderr` 打印输出调试语句。**

## 【线上故障】跟踪 执行慢/网络传输慢 的请求
在前一案例的基础上，将过滤条件修改一下，就可以做到其它效果。
+ 待修改代码
```awk
($6 >= second) {
   print $0
}
```

+ 跟踪：执行慢  
字段14`$upstream_response_time`代表的是后端服务器的执行时间，
```awk
($14 >= second) {
  print $0
}
```

+ 跟踪：网络传输慢  
`$request_time - $upstream_response_time`就是网络传输的时间
```awk
($6 - $14 >= second) {
  print $0
}
```

## 【线上故障】分析网络慢的请求
有些情况下，移动客户端（4G/3G网络) + 较大的响应体，就会产生比较多的网络消耗，拖累整体的响应时间。这个是需要定时做查询分析的，如果没有大数据，AWK也很方便。

分析网络传输和数据大小的关系：`ana-network.awk`

```awk
# awk -f ana-network.awk access.log
# awk -f ana-network.awk access.log -v 
function join(arr, sep, rst, i) {
  for (i = 1; i <= length(arr); i++)
    rst = rst (i > 1 ? sep : "") arr[i]

  return rst
}

function idx(sizes, size, i) {
  for (i in sizes)
    if (sizes[i] >= size)
      return i;

  return i;
}

BEGIN {
  FS = " [|]# "
  split(v_sizes ? v_sizes : "1024 2048 4196", sizes, "[ ;,]")

  printf "将使用这些大小区间来统计响应时间: %s \n\n", join(sizes, ",")  > "/dev/stderr"
}

{
  second = $6 - $14
  sizeIdx = idx(sizes, $7)

  if (maxTimes[sizeIdx] < second)
    maxTimes[sizeIdx] = second
  if (minTimex[sizeIdx] && (minTimes[sizeIdx] > second))
    minTimes[sizeIdx] = second

  counts[sizeIdx]++
  sums[sizeIdx] += second
}

END {
  _format = "%9s %9s %9s %9s %14s \n"
  printf _format, "size", "count", "max", "min", "avg"
  for (i = 1; i <= length(sizes) + 1; i++)
    printf _format,
      i <= length(sizes) ? " <= "sizes[i] : " > "sizes[i - 1],
      counts[i] ? counts[i] : 0,
      maxTimes[i] ? maxTimes[i] : 0,
      minTimes[i] ? minTime[i] : 0,
      counts[i] ? sums[i] / counts[i] : 0
}
```


## 线上日志大文件处理的注意事项
**一定要先测试**
**一定要先测试**
**一定要先测试**

这个是个人的经验教训：线上AWK处理大文件，一定要先测试。 

线上的机器，如果原本有一定的压力，一个大磁盘操作就可能是压倒骆驼的最后那根稻草。`awk`本身倒不是很严重，因为它最多只能占据一核CPU。`cat`带来的磁盘压力更可怕，在极端环境下，连锁反应就崩溃了。

比如这样： `cat access.log | head -10000 | awk -f ana.awk | more`

+ `cat`来获取文件，或`zcat`获取压缩文件
+ `head -10000`获取前面的若干行，这个在测试阶段必须要加，哪怕是写一百万一个亿，也比不写的好。
+ 最后的这个`more`是为了避免输出太多，到时候`ctrl + c`都来不及。  
个人非常熟练的情况下，这个环节可以忽略。  
使用`less`效果更好，因为它会进入到交互界面，支持查询。


## 附录

### access.log
这个是我们的部分nginx访问日志
```
157.255.131.183 |# - |# 2020-05-06 00:06:01 |# GET /bus/line!encRefreshLineDetail.action?stats_act=auto_refresh&refreshModule=&sign=MflpWbAMMtOhC%2B%2FLeOX8eA%3D%3D&language=1&cityId=014&transmission=&lineNo=M3853&phoneBrand=OPPO&gps_ts=1588688183510&lchsrc=icon&system_push_open=1&lat=22.545575&deviceType=OPPO+R11st&geo_type=gcj&o1=8b12a74508a73536d65afce26d88b16c107b2fc2&targetStationLng=113.96663137225858&cryptoSign=2d062170d1f4f8f450f02d0b27a3aa3a&lng=114.067892&first_src=app_oppo_store&vc=176&isNewLineDetail=1&gpstype=gcj&geo_lt=2&last_src=app_oppo_store&sstate=3&wifi_open=1&screenDensity=3.0&needAds=false&flpolicy=1&astate=1&bdStnLng=113.96009695847908&screenWidth=1080&cshow=other&nw=WIFI&stats_referer=searchHistory&lineName=M385&AndroidID=6fecf31956be2c36&mac=d4%3A1a%3A3f%3A5a%3Ab3%3A3d&bdStnLat=22.563166913371045&persistTravelId=&stationName=%E9%BE%99%E4%BA%95%E5%B7%A5%E4%B8%9A%E5%8C%BA&udid=9889906c-3280-49b0-9e0e-d26204d3fdfd&targetStationLat=22.568996038396172&direction=0&targetOrder=4&push_open=1&sv=8.1.0&geo_lac=550.0&screenHeight=1962&lineId=0755-M3853-0&showModule=1%2C1%3B2%2C1%3B3%2C1%3B4%2C1%3B&userAgent=Mozilla%2F5.0+%28Linux%3B+Android+8.1.0%3B+OPPO+R11st+Build%2FOPM1.171019.011%3B+wv%29+AppleWebKit%2F537.36+%28KHTML%2C+like+Gecko%29+Version%2F4.0+Chrome%2F62.0.3202.84+Mobile+Safari%2F537.36&userId=unknown&filter=1&paramsMakeUp=is&s=android&geo_lng=114.067892&geo_lat=22.545575&v=3.91.0&imei=868639038438756&stats_order=1-2&timeStamp=1588694762021&cryptoSignStr=c1ec5963f63c1608fadc1775284b58b5&dns_error=3 HTTP/1.1 |# 200 |# 0.040 |# 2676 |# - |# Dalvik/2.1.0 (Linux; U; Android 8.1.0; OPPO R11st Build/OPM1.171019.011) |# 120.229.52.6 |# api.chelaile.net.cn |# 172.17.17.49:8090 |# 200 |# 0.040 |# 158869476169224eca7f229150e5b1bd |# https |# -
172.17.16.121 |# - |# 2020-05-06 00:06:01 |# GET /bus/line!lineDetail.action?lineId=020-00601-1&direction=1&s=h5&v=3.3.19&sign=1&cityId=040&gpstype=gcj HTTP/1.1 |# 200 |# 0.020 |# 3883 |# http://web.chelaile.net.cn/ch5/index.html?utm_source=webapp_meizu_map&src=webapp_meizu_map&utm_medium=menu&hideFooter=1&gpstype=gcj&cityName=%E5%B9%BF%E5%B7%9E&cityId=040&supportSubway=1&cityVersion=0&lat=23.079135&lng=113.233398 |# Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.86 Safari/537.36 |# - |# api.chelaile.net.cn |# 172.17.16.150:8090 |# 200 |# 0.020 |# 158869476173176148dfc5dadced730f |# http |# -
183.40.217.253 |# - |# 2020-05-06 00:06:01 |# GET /api/bus/line!encryptedLineDetail.action?s=h5&wxs=wx_app&sign=1&v=3.8.156&src=weixinapp_cx&ctm_mp=mp_wx&cityId=040&userId=okBHq0NX9JkqLhp6NllM4maiohW4&h5Id=okBHq0NX9JkqLhp6NllM4maiohW4&lat=23.075265&lng=113.28792&geo_lat=23.075265&geo_lng=113.28792&gpstype=wgs&isNewCity=0&lineId=002031512162&targetOrder=16&cryptoSign=4debff6219f5a201e5c73f4f38681c7c HTTP/1.1 |# 200 |# 0.024 |# 10260 |# https://servicewechat.com/wx71d589ea01ce3321/379/page-frame.html |# Mozilla/5.0 (Linux; Android 9; Redmi K20 Pro Build/PKQ1.181121.001; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/74.0.3729.136 Mobile Safari/537.36 MicroMessenger/7.0.13.1640(0x27000D39) Process/appbrand4 NetType/4G Language/zh_CN ABI/arm64 WeChat/arm64 |# - |# web.chelaile.net.cn |# 172.17.16.19:80 |# 200 |# 0.024 |# https |# text
172.17.16.19 |# - |# 2020-05-06 00:06:01 |# GET /basesearch/client/clientSearch.action?s=h5&wxs=wx_app&sign=1&v=3.8.156&src=weixinapp_cx&ctm_mp=mp_wx&cityId=040&userId=okBHq0M7RNkwqFrLtia96iE16qlk&h5Id=okBHq0M7RNkwqFrLtia96iE16qlk&unionId=oSpTTjt3ei-o_zY3APX86wqPf1oA&lat=23.152059&lng=113.217306&geo_lat=23.152059&geo_lng=113.217306&gpstype=wgs&key=%E6%A7%8E%E5%A4%B4 HTTP/1.1 |# 200 |# 0.121 |# 807 |# https://servicewechat.com/wx71d589ea01ce3321/379/page-frame.html |# Mozilla/5.0 (Linux; Android 9; PAR-AL00 Build/HUAWEIPAR-AL00; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/79.0.3945.116 Mobile Safari/537.36 MicroMessenger/7.0.13.1640(0x27000D39) Process/appbrand2 NetType/4G Language/zh_CN ABI/arm64 WeChat/arm64 |# - |# api.chelaile.net.cn |# 172.17.16.69:9397 |# 200 |# 0.121 |# 15886947616688c4aaf5c89c56fc9117 |# http |# application/json
111.85.198.3 |# - |# 2020-05-06 00:06:01 |# GET /bus/stop!versionUpdtControl.action?mac=&userId=&geo_lng=106.776500&timestamp=1588694761753&gps_ts=1588694758836&accountId=32979580&sstate=3&appStartId=1588694752593_1717703&userAgent=Mozilla/5.0%20(iPhone;%20CPU%20iPhone%20OS%2013_4_1%20like%20Mac%20OS%20X)%20AppleWebKit/605.1.15%20(KHTML,%20like%20Gecko)%20Mobile/15E148&vc=15902&gps_speed=-1&sv=13.4.1&geo_lat=26.570035&screenWidth=1125&imsi=46000&geo_lac=65.000000&localCityId=083&secret=6b6adab1a48b41eeb24cd5bbfe71ff78&gpsAccuracy=65.000000&deviceType=iPhoneX&idfa=C5C7308F-6878-4747-8F6B-9FF8EFF66737&idfv=58D2FB5E-AE56-4876-AA7D-060829C45FAF&screenHeight=2436&localIdCreateTime=1588405573863&rnver=5.90.2.16&sign=uZKiBqWIaewibCdqc78IUQ==&s=IOS&cityId=003&dpi=3&push_open=1&astate=1&wifi_open=1&locState=3&gps_accuracy=65&v=5.90.2&geo_type=wgs&nw=WiFi&language=1&vendor=apple&lchsrc=icon&udid=110b389dd2b86d5aae8ae31d145584047217a462&locOn=1 HTTP/1.1 |# 200 |# 0.003 |# 85 |# - |# lite/5.90.2 (iPhone; iOS 13.4.1; Scale/3.00) |# - |# api.chelaile.net.cn |# 172.17.17.84:8090 |# 200 |# 0.003 |# 1588694761796794d759e51cd04eaddf |# https |# -
183.202.39.142 |# - |# 2020-05-06 00:06:01 |# GET /bus/reminder!updateToken.action?last_src=app_oppo_store&s=android&push_open=1&token=73beb18f06338d74a8c7acf1586a4a5a&userId=unknown&astate=1&vc=176&sv=6.0.1&v=3.91.0&imei=864412039047636&system_push_open=1&udid=1156782f-7bae-4443-a3a6-ab73dcbd5be5&cityId=094&sign=MflpWbAMMtMTGBI10KL38g%3D%3D&wifi_open=1&mac=98%3A6f%3A60%3Acd%3A20%3Aa5&deviceType=OPPO+R9sk&lchsrc=icon&nw=WIFI&sstate=3&paramsMakeUp=is&AndroidID=b47a3dc850c07564&phoneBrand=OPPO&language=1&first_src=app_oppo_store&tokenType=0 HTTP/1.1 |# 200 |# 0.001 |# 77 |# - |# Dalvik/2.1.0 (Linux; U; Android 6.0.1; OPPO R9sk Build/MMB29M) |# - |# api.chelaile.net.cn |# 172.17.17.18:9090 |# 200 |# 0.001 |# 158869476181379ce549a22e8a36d369 |# https |# -
121.29.84.125 |# - |# 2020-05-06 00:06:01 |# GET /wow/user!query2.action?last_src=app_360_sj&s=android&push_open=1&userId=unknown&geo_lt=4&geo_lat=38.141958&astate=1&vc=162&sv=7.1.1&v=3.86.2&secret=62c0accde2f94d8da5f4d97e26a10a5f&imei=99000788013541&oaIdDeviceCode=1008611&system_push_open=1&udid=5a82f277-2495-485d-911f-c10c5a215523&oaIdDeviceSupport=notSupport&cityId=053&sign=MflpWbAMMtP1OS32RR%2FzsA%3D%3D&geo_type=gcj&deviceType=VCR-A0&mac=54%3Adc%3A1d%3Ab6%3Ae1%3A14&wifi_open=1&lchsrc=icon&sstate=3&nw=WIFI&paramsMakeUp=is&AndroidID=58af64db77839869&geo_lac=100.0&phoneBrand=Coolpad&accountId=86987530&oaId=unknown&language=1&first_src=app_qq_sj&geo_lng=114.586532 HTTP/1.1 |# 200 |# 0.022 |# 582 |# - |# Dalvik/2.1.0 (Linux; U; Android 7.1.1; VCR-A0 Build/VCRSCN00X1000MPX1711140) |# - |# api.chelaile.net.cn |# 172.17.17.100:9070 |# 200 |# 0.022 |# 158869476180911f1c2bb8903b917cff |# https |# -
119.123.206.62 |# - |# 2020-05-06 00:06:01 |# GET /sensors?project=default&token=fae3b5aa4801542a38bad43ba8f4f34d631ac3789c5475a4b5e065def990c68d&data=eyJkaXN0aW5jdF9pZCI6Im9rQkhxMEl3Qjlfb0ljbHVOektrNzhzVUc3eDQiLCJsaWIiOnsiJGxpYiI6Ik1pbmlQcm9ncmFtIiwiJGxpYl9tZXRob2QiOiJjb2RlIiwiJGxpYl92ZXJzaW9uIjoiMC42In0sInByb3BlcnRpZXMiOnsiJGxpYiI6Ik1pbmlQcm9ncmFtIiwiJGxpYl92ZXJzaW9uIjoiMC42IiwiJHVzZXJfYWdlbnQiOiJTZW5zb3JzQW5hbHl0aWNzIE1QIFNESyIsIiRuZXR3b3JrX3R5cGUiOiJ3aWZpIiwiJG1vZGVsIjoiaVBob25lIDc8aVBob25lOSwxPiIsIiRzY3JlZW5fd2lkdGgiOjM3NSwiJHNjcmVlbl9oZWlnaHQiOjYwMywiJG9zIjoiaU9TIiwiJG9zX3ZlcnNpb24iOiIxMi40LjEiLCJhcHBfbmFtZSI6Iui9puadpeS6hueyvuWHhuWunuaXtuWFrOS6pCIsImFwcF92ZXJzaW9uIjoiMy44LjE1NiIsImN1clBhZ2UiOiJob21lIiwiZGVzY3JpYiI6ImhvbWUiLCJzcmMiOiJ3ZWl4aW5hcHBfY3giLCJ1c2VyVHlwZSI6IndlY2hhdF9tcCIsInBsYXRmb3JtIjoiaW9zIiwiY2l0eSI6Iua3seWcsyIsImdlb19sbmciOiIxMTMuODY5MjE2OTE4OTQ1MzEiLCJnZW9fbGF0IjoiMjIuNTgzNDE5Nzk5ODA0Njg4IiwidXNlcklkIjoib2tCSHEwSXdCOV9vSWNsdU56S2s3OHNVRzd4NCIsIiRpc19maXJzdF9kYXkiOmZhbHNlfSwidHlwZSI6InRyYWNrIiwiZXZlbnQiOiJXRUNIQVRfTVBfUEFHRSIsIl9ub2NhY2hlIjoiMzM0MTQ4ODMzMTA0MyJ9 HTTP/1.1 |# 200 |# 0.001 |# 31 |# https://servicewechat.com/wx71d589ea01ce3321/379/page-frame.html |# Mozilla/5.0 (iPhone; CPU iPhone OS 12_4_1 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15E148 MicroMessenger/7.0.12(0x17000c2d) NetType/WIFI Language/zh_CN |# - |# web.chelaile.net.cn |# 172.17.17.73:8106 |# 200 |# 0.001 |# https |# application/json
120.230.99.30 |# - |# 2020-05-06 00:06:01 |# GET /bus/reminder!updateToken.action?last_src=app_oppo_store&s=android&push_open=1&token=23cc4fe44ab8c81ccc5a5abbf0cbbe8f&userId=unknown&astate=1&vc=176&sv=6.0.1&v=3.91.0&secret=7a8236af24cc47ecb1fd84d566648d2a&imei=864315038933153&system_push_open=1&udid=91b7d1d2-5ef7-480b-add6-525eeff7748f&cityId=040&sign=MflpWbAMMtMkDPmv7pLXxA%3D%3D&wifi_open=1&mac=d4%3A50%3A3f%3A50%3A0e%3Add&deviceType=OPPO+R9sk&lchsrc=icon&nw=WIFI&sstate=3&paramsMakeUp=is&AndroidID=fcfa1a1616330e7&accountId=58143812&phoneBrand=OPPO&language=1&first_src=app_oppo_store&tokenType=0 HTTP/1.1 |# 200 |# 0.001 |# 77 |# - |# Dalvik/2.1.0 (Linux; U; Android 6.0.1; OPPO R9sk Build/MMB29M) |# - |# api.chelaile.net.cn |# 172.17.17.103:9090 |# 200 |# 0.001 |# 15886947618392e5366b12e672bdeab2 |# https |# -
112.96.118.28 |# - |# 2020-05-06 00:06:01 |# GET /bus/reminder!updateToken.action?last_src=app_oppo_store&s=android&push_open=1&token=01f582fb46ad1e9bd47c2eabf93f42c7&userId=unknown&astate=1&vc=175&sv=6.0.1&v=3.90.4&secret=8e38bc377a2d47cb8d2964f0a7ba80ea&imei=866491036515611&system_push_open=1&udid=741f2e45-5624-4e86-8364-0c44025231a5&cityId=019&sign=MflpWbAMMtM95KYtWgrN6Q%3D%3D&wifi_open=0&mac=30%3A84%3A54%3A6c%3A58%3A17&deviceType=OPPO+A57&lchsrc=icon&nw=MOBILE_LTE&sstate=3&paramsMakeUp=is&AndroidID=91b826d010703913&accountId=133077807&phoneBrand=OPPO&language=1&first_src=app_oppo_store&tokenType=0 HTTP/1.1 |# 200 |# 0.002 |# 77 |# - |# Dalvik/2.1.0 (Linux; U; Android 6.0.1; OPPO A57 Build/MMB29M) |# - |# api.chelaile.net.cn |# 172.17.17.18:9090 |# 200 |# 0.002 |# 1588694761847388a64bb9e328cee8d1 |# https |# -
```
