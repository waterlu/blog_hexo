---
title: 性能调优
date: 2018-05-03 17:40:28
tags:
description: TODO
---

## Linux性能调优

### CPU占用过高

首先，通过top -c指令查看进程的cpu占用情况，找到cpu占用高的进程pid；

然后，通过top -Hp [pid]指令查看进程内线程运行情况，找出cpu占用高的线程pid；

```shell
root@iZ2ze4k1o3ish3waeplqi8Z:~# top -Hp 14933
PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND                   14965 root      20   0 2449044 493880  15588 S  0.3 24.1   0:05.92 java                   14982 root      20   0 2449044 493880  15588 S  0.3 24.1   0:02.76 java                   14984 root      20   0 2449044 493880  15588 S  0.3 24.1   0:18.18 java                   14985 root      20   0 2449044 493880  15588 S  0.3 24.1   0:02.45 java                   14999 root      20   0 2449044 493880  15588 S  0.3 24.1   0:13.40 java
```

假设第一个线程14965占用cpu最多，将[pid]=14965转换成16进制`0x3a75` 

```shell
root@iZ2ze4k1o3ish3waeplqi8Z:~# printf "%x\n" 14965
3a75
```

最后，打印进程堆栈，过滤线程pid，查看线程栈，找出执行任务：

```shell
root@iZ2ze4k1o3ish3waeplqi8Z:~# jstack 14933 | grep '0x3a75' -C5 --color
"nioEventLoopGroup-2-1" #27 prio=10 os_prio=0 tid=0x00007fe5b890b800 nid=0x3a75 runnable [0x00007fe587802000]
   java.lang.Thread.State: RUNNABLE
	at sun.nio.ch.EPollArrayWrapper.epollWait(Native Method)
	at sun.nio.ch.EPollArrayWrapper.poll(EPollArrayWrapper.java:269)
	at sun.nio.ch.EPollSelectorImpl.doSelect(EPollSelectorImpl.java:93)
	at sun.nio.ch.SelectorImpl.lockAndDoSelect(SelectorImpl.java:86)
```

> 本例为虚拟例子，所以当前执行的任务是poll()。

### 端口占用

netstat -lap | grep 8051

lsof -i :8051

```java
root@iZ2ze4k1o3ish3waeplqi8Z:~# netstat -lap | grep 8051
tcp        0      0 *:8051                  *:*                     LISTEN      14933/java      
tcp        0      0 172.17.134.10:8051      172.17.134.96:55014     ESTABLISHED 14933/java      
tcp        0      0 172.17.134.10:8051      172.17.134.96:51712     ESTABLISHED 14933/java      
tcp        0      0 172.17.134.10:8051      172.17.134.9:56656      FIN_WAIT2   -               
tcp        0      0 172.17.134.10:8051      172.17.134.9:56652      ESTABLISHED 14933/java      
tcp        0      0 172.17.134.10:8051      172.17.134.11:44974     ESTABLISHED 14933/java 

root@iZ2ze4k1o3ish3waeplqi8Z:~# lsof -i:8051
COMMAND   PID USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
java    14933 root   35u  IPv4 43755096      0t0  TCP 172.17.134.10:8051->172.17.134.9:56652 (ESTABLISHED)
java    14933 root   36u  IPv4 43719121      0t0  TCP *:8051 (LISTEN)
java    14933 root   48u  IPv4 43719551      0t0  TCP 172.17.134.10:8051->172.17.134.96:51712 (ESTABLISHED)
java    14933 root   59u  IPv4 43751694      0t0  TCP 172.17.134.10:8051->172.17.134.96:55014 (ESTABLISHED)

```



```shell
jmap -heap 14933
jmap -histo:live 14933 | more
ll /proc/14933/fd/
ll /proc/14933/task/
ll /proc/14933/task | wc -l
```

