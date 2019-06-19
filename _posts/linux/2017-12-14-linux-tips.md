---
title:  "Linux Tips"
date:   2018-12-19 07:08:11 +0800
categories: linux tips
tags: Linux Tips
is_tip: true
---


## send udp packet
```bash
$ echo "This is my data" > /dev/udp/127.0.0.1/3000
```

## nload查看网络实时流量
个人知道的感觉最好的

## zmore
```bash
$ zmore nginx1.20171224.gz | awk -F ' [|]# ' '{if ($3 >=; "2017-12-22 09:00:00" ) exit; if ($3 >= "2017-12-22 09:00:00") print $0;}' | more
```


## gzip
```
gzip filename
  -f force to overwrite existing gz file
  -k keep original file
  -r folder recurve
  -1  compress level1
```

```
depress
gzip  gzfile
  -d depress
  -l list members in
  -v valid
```


## redirect err to info
`2>&1`

``` bash
$ 2>&1 nginx -V
$ nginx -V 2>&1
```
prefer to the former.


## nohup

```bash
$ nohup node server.js > nohup.log 2>&1 &
```


## ps
```bash
$ ps -p $PPID -o cmd | tail -1
```


## tar
```
tar --extract --file={tarball.tar} {file}
```


## check swap memory
```bash
$ awk '/VmSwap|Name/{printf $2 " " $3}END{ print  "" }'  /proc/[pid]/status
```
```bash
for file in /proc/*/status ; do
  awk '/VmSwap|Name/{printf $2 " " $3}END{ print ""}' $file;
done
```
```bash
for es in $(jps | grep Elasticsearch | awk '{print $1}'); do
  awk '/VmSwap|Name/{printf $2 " " $3}END{ print " " }'  /proc/$es/status;
done
```


## cannot ssh to remote machine with root
1. modify /etc/ssh/sshd_config
```vim
#PermitRootLogin prohibit-password
PermitRootLogin yes
```

2. restart ssh service
```bash
$ service ssh restart
```


## prevent ssh from disconnecting
`~/.ssh/config`

```vim
Host *
  ServerAliveInterval 60
```


## System is booting up. See pam_nologin

remove `/var/tun/nologin` `/etc/nologin`


## set language support

+ `export LC_ALL=en_US.UTF-8`
or
+ `export LANG=en_US.UTF-8`


## add language package for linux:
https://perlgeek.de/en/article/set-up-a-clean-utf8-environment
```bash
$ dpkg-reconfigure locales

(Enter the items you want to select, separated by spaces.)
Locales to be generated: 149 471

Many packages in Debian use locales to display text in the correct language for the user. You can choose a default
locale for the system from the generated locales.
This will select the default language for the entire system. If this system is a multi-user system where not all
users are able to speak the default language, they will experience difficulties.
  1. None  2. C.UTF-8  3. en_US.UTF-8  4. zh_CN.UTF-8
Default locale for the system environment: 3
```

## no space in /boot

```bash
$ dpkg -l | grep linux-image
$ dpkg -l | grep linux-headers

$ apt-get purge linux-image-2.6.31-17-server
$ dpkg --purge linux-image-2.6.28-11-server

$ apt-get autoremove
```

## netstat
```bash
$ netstat -atulpn
```

## find

### -o more arguments
```bash
$ find . -name "java" -o -name "resources"
```

### -not
```bash
$ find . -not -name "java"
```

### \\(\\)
```bash
$ find . -not \( -name "java" -o -name "resources" \)
```


## test linux interactive mode
```bash
$ [[ $- == *i* ]] && echo 'Interactive' || echo 'Not interactive'
```
