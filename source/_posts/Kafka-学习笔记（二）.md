---
title: Kafka 学习笔记（二）
date: 2018-05-03 22:57:00
author: ACE NI
tags:
  - kafka
categories:
  - 高并发架构
---
# kafka核心组件

主要包括延长操作组件、控制器、协调器、网络通信、日志管理器、副本管理器、动态配置管理器及心跳检测等。

<!-- more -->

## 一、延长操作组件

### DelayedOperation

一个基于事件启动有失效时间的TimerTask。