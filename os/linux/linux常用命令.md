--- 
title: linux常用命令 
date: 2019-06-03
categories: 
- linux
---
# top

```c
任务队列信息
当前系统时间 系统累计运行时间 当前登录用户 活跃进程数1，5，15分钟
top - 11:47:19 up 298 days, 19:59,  1 user,  load average: 0.99, 1.01, 1.03
进程信息
总共进程数 正在运行 休眠进程 停止进程 僵尸进程
Tasks: 175 total,   4 running, 170 sleeping,   1 stopped,   0 zombie
CPU信息
用户态占用CPU百分比 内核态占用CPU百分比 改变过优先级的进程百分比 空闲百分比 IO等待百分比 硬中断 软中断
%Cpu(s): 15.9 us, 35.1 sy,  0.0 ni, 49.1 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
内存状态
物理内存总量 空闲内存总量 使用内存总量 缓存的内存量
KiB Mem :  8009832 total,   225372 free,   662292 used,  7122168 buff/cache
swap分区信息
交换区总量 空闲 使用
KiB Swap:        0 total,        0 free,        0 used.  6916720 avail Mem

进程ID
      用户
                优先级
                    nice值 负数表示高优先级 整数表示低优先级
                          进程使用的虚拟内存总量
                                  进程使用的，未被换出的物理内存大小
                                         共享内存大小
                                             进程状态 D=不可中断的睡眠状态 R=运行 S=睡眠 T=跟踪/停止 Z=僵尸进程
                                                上次更新到现在CPU占用比
                                                     进程使用的物理内存百分比
                                                              进程使用的CPU时间总计，单位1/100秒
                                                                    命令
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 1047 root      20   0  110088    620    488 R 100.0  0.0 126488:57 agetty
 8556 root      20   0  572548  22072   2776 S   0.3  0.3 348:57.12 YDService
17633 root      20   0  611276  13908   2368 S   0.3  0.2  16:25.61 barad_agent
29331 nber1994  20   0  162020   2344   1604 R   0.3  0.0   0:00.01 top
    1 root      20   0  191088   3492   2020 S   0.0  0.0  28:13.70 systemd
    2 root      20   0       0      0      0 S   0.0  0.0   0:03.02 kthreadd
    3 root      20   0       0      0      0 S   0.0  0.0   0:33.11 ksoftirqd/0
    5 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/0:0H
    7 root      rt   0       0      0      0 S   0.0  0.0   0:17.95 migration/0
    8 root      20   0       0      0      0 S   0.0  0.0   0:00.00 rcu_bh
    9 root      20   0       0      0      0 R   0.0  0.0 116:01.36 rcu_sched
   10 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 lru-add-drain
   11 root      rt   0       0      0      0 S   0.0  0.0   1:56.85 watchdog/0
   12 root      rt   0       0      0      0 S   0.0  0.0   1:30.75 watchdog/1
   13 root      rt   0       0      0      0 S   0.0  0.0   0:23.45 migration/1
   14 root      20   0       0      0      0 S   0.0  0.0   0:52.02 ksoftirqd/1
   16 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/1:0H
   18 root      20   0       0      0      0 S   0.0  0.0   0:00.00 kdevtmpfs
   19 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 netns
   20 root      20   0       0      0      0 S   0.0  0.0   0:08.81 khungtaskd
   21 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 writeback
   22 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kintegrityd
   23 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 bioset
   24 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kblockd
   25 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 md
   26 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 edac-poller
   32 root      20   0       0      0      0 S   0.0  0.0   0:15.16 kswapd0
   33 root      25   5       0      0      0 S   0.0  0.0   0:00.00 ksmd
   35 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 crypto
   43 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kthrotld
   45 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kmpath_rdacd
   46 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kaluad
   47 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kpsmoused
   48 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 ipv6_addrconf
   62 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 deferwq
   98 root      20   0       0      0      0 S   0.0  0.0   0:09.52 kauditd
```
* 第四行中使用中的内存总量（used）指的是现在系统内核控制的内存数，空闲内存总量（free）是内核还未纳入其管控范围的数量。纳入内核管理的内存不见得都在使用中，还包括过去使用过的现在可以被重复利用的内存，内核并不把这些可被重新使用的内存交还到free中去，因此在linux上free内存会越来越少，但不用为此担心。

* 如果出于习惯去计算可用内存数，这里有个近似的计算公式：第四行的free + 第四行的buffers + 第五行的cached，按这个公式此台服务器的可用内存：18537836k +169884k +3612636k = 22GB左右。

* 对于内存监控，在top里我们要时刻监控第五行swap交换分区的used，如果这个数值在不断的变化，说明内核在不断进行内存和swap的数据交换，这是真正的内存不够用了。

![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20190602121147976_262344756.png)
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20190602121309965_444386801.png)
