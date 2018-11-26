---
title: linux性能学习
date: 2018-11-24 22:20:16
author: ACE NI
tags:
  - 平均负载
categories:
  - Linux
---
# 性能学习基础
## 性能指标是什么
**高并发  --  吞吐 **
**响应快 -- 延时 **

![linux-resourse](https://ws2.sinaimg.cn/large/006tNbRwgy1fxjk2tqyrbj30t70gjjsj.jpg)

<!-- more -->

### Linux 性能工具图谱
![linux-tools](https://ws2.sinaimg.cn/large/006tNbRwgy1fxjjf7g6wej316v0u01i5.jpg)

### linux性能思维导图
![linux性能优化](https://ws4.sinaimg.cn/large/006tNbRwgy1fxjjljbisfj30u02q7grl.jpg)

## 平均负载
单位时间内，系统处于**可运行状态**和**不可中断状态**的平均进程数，也就是**平均活跃进程数**。
- 可运行状态进程：正在使用CPU或者正在等待CPU的进程 -- 常用ps命令可以看到的，处于R状态(Running 或 Runable) 的进程。
- 不可中断状态进程：正处于内核态关键流程中的进程，并且这些流程是不可打断的。最常见的是等待硬件设备的I/O响应，常用ps命令看到的，处于D状态的进程。

**当平均负载高于CPU数量70%的时候**，就应该分析排查负载高的问题了。

*平均负载 ≠ CPU使用率 *
- CPU密集型进程，使用大量CPU会导致平均负载升高，CPU使用率也高
- I/O密集型进程，等待I/O也会导致平均负载升高，但CPU使用率不一定高
- 大量等待CPU的进程调度也会导致平均负载和CPU使用率升高

### 压力分析

stress： linux系统压力测试工具。
sysstat：包含常用的linux性能工具，用来监控和分析系统性能。包含 `mpstat`和`pidstat`两个命令。
- `mpstat` 用来实时查看每个 CPU 的性能指标，以及所有 CPU 的平均指标
- `pidstat` 用来实时查看进程的 CPU、内存、I/O 以及上下文切换等性能指标

```bash
root$ uptime
11:17:07 up 264 days, 14:21,  4 users,  load average: 0.06, 0.12, 0.13
```

#### CPU 密集型进程
```bash
#第一个终端运行stress命令，模拟一个CPU使用率100%的场景
root$ stress --cpu 1 --timeout 600
stress: info: [20102] dispatching hogs: 1 cpu, 0 io, 0 vm, 0 hdd

#第二个终端运行uptime查看平均负载
# -d 参数表示高亮显示变化的区域
root$ watch -d uptime
11:24:38 up 264 days, 14:28,  4 users,  load average: 1.08, 0.67, 0.3
# 一分钟的平均负载已慢慢升到1.0.8

#第三个终端运行mpstat查看CPU使用情况
# -P ALL 表示监控所有 CPU，后面数字 5 表示间隔5秒输出一组数据
root$ mpstat -P ALL 5
Linux 3.10.0-693.2.2.el7.x86_64 (TV-DEV-DB01) 	2018年11月25日 	_x86_64_	(4 CPU)

11时20分51秒  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
11时20分56秒  all   25.66    0.00    0.35    0.55    0.00    0.05    0.00    0.00    0.00   73.38
11时20分56秒    0    0.80    0.00    0.40    0.00    0.00    0.20    0.00    0.00    0.00   98.59
11时20分56秒    1    0.81    0.00    0.40    2.02    0.00    0.00    0.00    0.00    0.00   96.77
11时20分56秒    2    0.60    0.00    0.40    0.00    0.00    0.20    0.00    0.00    0.00   98.80
11时20分56秒    3   99.80    0.00    0.00    0.00    0.00    0.20    0.00    0.00    0.00    0.00

```
看到一个CPU的使用率为100%，但它的iowait只有0。
```bash
# 间隔5秒输出一组数据
root$ pidstat -u 5 1
Linux 3.10.0-693.2.2.el7.x86_64 (TV-DEV-DB01) 	2018年11月25日 	_x86_64_	(4 CPU)

11时32分23秒   UID       PID    %usr %system  %guest    %CPU   CPU  Command
11时32分28秒     0       940    0.20    0.20    0.00    0.40     3  AliYunDun
11时32分28秒     0      6164    0.40    0.00    0.00    0.40     2  java
11时32分28秒     0      6483    0.00    0.20    0.00    0.20     1  aliyun-service
11时32分28秒     0     14754    0.20    0.00    0.00    0.20     3  java
11时32分28秒     0     20104    0.20    0.20    0.00    0.40     3  watch
11时32分28秒     0     20826  100.00    0.00    0.00  100.00     1  stress
11时32分28秒     0     25477    0.20    0.40    0.00    0.60     2  java
11时32分28秒     0     25718    0.20    0.00    0.00    0.20     1  java
11时32分28秒     0     28889    1.00    0.60    0.00    1.60     1  java
11时32分28秒     0     30891    0.20    0.20    0.00    0.40     0  mongod
11时32分28秒     0     32076    0.00    0.20    0.00    0.20     3  redis-server
```
可以看到，stress进程的CPU使用率为100%。

#### I/O密集型进程
```bash
#第一个终端运行stress命令，模拟一个I/O压力的场景
root$ stress -i 1 --timeout 600

#第二个终端运行uptime查看平均负载
root$ watch -d uptime

#第三个终端运行mpstat查看CPU使用情况
root$ mpstat -P ALL 5
Linux 3.10.0-693.2.2.el7.x86_64 (TV-DEV-DB01) 	2018年11月25日 	_x86_64_	(4 CPU)

11时58分45秒  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
11时58分50秒  all    0.86    0.00   16.56    5.50    0.00    0.00    0.00    0.00    0.00   77.08
11时58分50秒    0    0.40    0.00   52.92   10.06    0.00    0.00    0.00    0.00    0.00   36.62
11时58分50秒    1    0.40    0.00    1.01    1.82    0.00    0.00    0.00    0.00    0.00   96.77
11时58分50秒    2    2.03    0.00   10.14    9.53    0.00    0.00    0.00    0.00    0.00   78.30
11时58分50秒    3    0.80    0.00    1.61    0.80    0.00    0.20    0.00    0.00    0.00   96.58
```
使用`pidstat`查看，会发现还是stress进程导致的。
使用`top`命令查看
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fxk6oqc6lmj315y0fg75f.jpg)

#### 大量进程的场景
当系统中运行的进程超出CPU运行能力时，就会出现等待CPU的进程。
```bash
root$ stress -c 16 --timeout 600
# 系统只有4个CPU，少于16个进程

root$ watch -d uptime
12:06:15 up 264 days, 15:10,  4 users,  load average: 14.90, 5.78, 2.35
```
![](https://ws3.sinaimg.cn/large/006tNbRwgy1fxk6inbdihj30w80quabu.jpg)

使用`top`命令查看
![](https://ws1.sinaimg.cn/large/006tNbRwgy1fxk6kwme8dj317q0twgo4.jpg)
