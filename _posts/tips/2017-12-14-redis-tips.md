---
title:  "Redis Tips"
date:   2018-12-19 07:08:11 +0800
categories: tips redis
tags: Redis Tips
---


## url for jedis config
``` yml
url : redis://localhost:6379/
url : any_name_or_empty:password@redis://localhost:6379/
```


## count the keys
```
EVAL "local ks,ret = redis.call('keys', ARGV[1]), 0; for k,v in ipairs(ks) do ret = ret + 1; end; return ret " 0  k*
```


## clear the keys
```
EVAL "local ks,ret = redis.call('keys', ARGV[1]), 0; for k,v in ipairs(ks) do redis.call('del', v); ret = ret + 1; end; return ret " 0  k*
```

## find big keys
```bash
redis-cli --bigkeys
```


## check clients
```
# echo "info clients
>  quit" | ncat localhost 6379
```
result is:
```bash
$110
# Clients
connected_clients:29
client_longest_output_list:0
client_biggest_input_buf:5
blocked_clients:0

+OK
```

+ 网厅

```bash
for ip4 in {174..176}; do
  ip="10.142.164.$ip4";
  for p in 6379; do
    echo $ip:$p $(printf "info clients\n quit\n" | ncat $ip $p | grep connected | awk 'BEGIN{FS=":"}{print $2}') ;
  done;
done
```
+ 登陆

```
for ip4 in {12..15}; do
  ip="10.20.48.$ip4";
  for p in 6376 {6380..6387}; do
    echo $ip:$p $(printf "info clients\n quit\n" | ncat $ip $p | grep connected | awk 'BEGIN{FS=":"}{print $2}') ;
  done;
done
```
[link](#count-the-keys)
