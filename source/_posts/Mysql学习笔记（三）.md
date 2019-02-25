---
title: MySQL学习笔记（三）
date: 2018-11-26 22:13:08
tags:
 - 数据库锁
 - 数据库事务
categories:
 - MySQL
---

# 全局锁

加全局读锁的命令：`Flush tables with read lock` (`FTWRT`)。

整个库处于只读状态，阻塞数据更新语句（数据的增删改）、数据定义语句（包括建表、修改表结构等）和更新类事务的提交语句。

**适用于不支持事务的引擎做全库逻辑备份的场景**

InnoDB做全库备份的方法： 官方自动的逻辑备份工具`mysqldump`使用参数`-single-transaction`，在**可重复读**隔离级别下开启一个事务，确保那个一致性视图。

<!-- more -->

# 表级锁

## 表锁

加表锁的命令：`lock tables ... read/write`。

表锁是最常用的处理并发的方式，但对于InnoDB这种支持行锁的引擎，一般不使用`lock tables`来控制并发。



## 元数据锁（MDL）

`MDL(metadata lock)`不需要显示使用，在访问一个表的时候会被自动加上。

- 当对一个表做增删改查操作的时候，加MDL读锁
- 当要对表做结构变更操作的时候，加MDL写锁
- 读锁之间不互斥，可以用多个线程同时对一张表增删改查
- 读写锁之间，写锁之间是互斥的，用来保证变更表结构操作的安全性



长事务导致改表数据库崩溃问题

![mysqlMDL](https://ws3.sinaimg.cn/large/006tNbRwgy1fxlukvpyxoj30vq0n1dh6.jpg)

A存在长事务，`session C`的MDL写锁互斥会block住，后续的读锁也会因为`session C`的写锁阻塞住。



因此必须解决长事务，事务不提交，就会一直占着MDL锁。

但对于热点表，会不停有新的请求，无法kill掉事务，可以在`alter table`语句里设定等待时间，如果在这个指定的等待时间里面能够拿到MDL写锁最好，拿不到也不要阻塞后面的业务语句，先放弃。

```mysql
ALTER TABLE tbl_name NOWAIT add column ...
ALTER TABLE tbl_name WAIT N add column ... 
```

