---
title: MySQL学习笔记（一）
date: 2018-11-25 12:19:45
tags:
 - mysql
categories:
 - 高并发架构
---

# MySQL基础架构
![mysql](https://ws2.sinaimg.cn/large/006tNbRwgy1fxkb71ysxaj31400u0428.jpg)

<!-- more -->

## 连接器
连接器负责跟客户端建立连接、获取权限、维持和管理连接。
```
mysql -h$ip -P$port -u$user -p
```
连接完成后，如果没有后续的操作，这个连接就处于空闲状态。客户端如果太长时间没动静，连接器就会自动断开连接，默认时长8小时，由`wait_timeout`参数控制。

建立连接的过程通常比较复杂，TCP握手，权限认证等，因此使用中尽量减少建立连接的动作，即尽量使用长连接而不是短连接。

*注意：大量使用长连接，可能导致内存占用太大，被系统强行杀掉(OOM)*

## 查询缓存
MySQL会以键值对的形式直接缓存到内存中，key是查询语句，value是查询的结果。
**通常不建议使用查询缓存，因为缓存失效非常频繁，总要对一个表更新，这个表上的所有查询缓存都会清空，更新压力大的数据库，查询缓存的命中率非常低**

设置参数`query_cache_type = DEMAND`，这样对于默认的SQL语句都不使用查询缓存。
实际开发用`mybatis`的话会用到一级缓存和二级缓存，并不需要使用MySQL的查询缓存。

*Spring又会使mybatis的一级缓存失效，二级缓存又存在弊端，所以使用时一定要结合实际场景。*

*注意：MySQL 8.0版本直接将查询缓存的整块功能删除了*

## 分析器
分析器会做”词法分析“和”语法分析“。
如当执行语句`select * from T where K = 1;`报不存在这个列，就是分析器阶段的处理结果。

## 优化器
优化器是在表里面有多个索引的时候，决定使用哪个索引；或者在一个语句有多表关联(join)的时候，决定各个表的连接顺序。

## 执行器
开始执行的时候，会先判断一下对表是否有执行权限，如果没有，就会返回没有权限的错误。如果有，就打开表调用引擎接口进行执行。

# MySQL日志模块

## redo log
当有一条记录需要更新的时候，InnoDB引擎就会先把记录写到redo log里，并更新到内存，等系统比较空闲的时候，InnoDB引擎会将这个操作记录更新到磁盘里。

这个操作使得InnoDB可以保证即使数据库发生异常重启，之前提交的记录也不会丢失，即`crash-safe`能力。

**redo log 是 InnoDB引擎特有的，是物理日志，记录了”在某个数据页做了什么操作。“**

*redo log 是循环写的，通过write pos 记录当前记录位置，checkpoint记录擦除位置。当write pos 追上 checkpoint时，停下来擦除部分记录。*

![redolog](https://ws2.sinaimg.cn/large/006tNbRwgy1fxkdlh9vxkj30vq0i8mxv.jpg)

`innodb_flush_log_at_trx_commit`参数设置为1，表示每次事务的redo log都直接持久化到磁盘。


## binlog
binlog 是MySQL server层实现的，所有引擎都可以使用。记录的是逻辑日志。分两张模式，statement格式的话是记SQL语句，row格式会记录行的内容，记两条，更新前和更新后都有。

`sync_binlog`参数设置为1，表示每次事务的binlog都持久化到磁盘。

由于binlog日志只能用于归档，InnoDB结合redo log 来实现 crash-safe 能力。

![update](https://ws4.sinaimg.cn/large/006tNbRwgy1fxkdrx3wtdj30u013zwgx.jpg)

图中浅色框表示在InnoDB内部执行，深色框表示是在执行器中执行的。

1、 prepare阶段   2、 写binlog   3、commit
当在2之前崩溃时
重启恢复redo log发现没有commit，binlog没有，事务回滚。
当在3之前崩溃时
重启恢复redo log发现没有commit，binlog有，认可事务，自动提交。

## 数据库还原
- 首先找到最近的一次全量备份，恢复到临时库。
- 从备份的时间点开始，依次重放binlog记录到需要还原的时间点

# MySQL事务
## 事务隔离级别
SQL 标准的事务隔离级别
1. 读未提交(read uncommitted)
2. 读提交(read committed)
3. 可重复读(repeatable read)
4. 串行化(serializabel)

```mysql
mysql> create table T(c int) engine=InnoDB;
insert into T(c) values(1);
```
![mysql事务](https://ws3.sinaimg.cn/large/006tNbRwgy1fxkkk5bhfvj30u011baca.jpg)

### 读未提交
一个事务还没提交时，它做的变更就能被别的事务看到。
若隔离级别是“读未提交”， 则 V1 的值就是 2。这时候事务B虽然还没有提交，但结果已经被A看到了。因此，V2、V3也都是2

### 读提交
一个事务提交之后，它做的变更才会被其他事务看到。
若隔离级别是”读提交“，则V1是1，V2的值是2。事务B的更新在提交后才能被A看到。所以，V3的值也是2。

### 可重复读
一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据一致。当然在可重复读隔离级别下，未提交变更对其他事务也是不可见的。
若隔离级别是”可重复读“，则V1、V2是1，V3是2。之所以V2还是1，遵循的就是事务在执行期间看到的数据前后必须是一致的。

### 串行化
对于同一行记录，”写“会加”写锁“，”读“会加”读锁”。当出现读写锁冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行。
若隔离级别是“串行化”，则在事务B执行“将1改成2”的时候，会被锁住。直到事务A提交后，事务B才可以继续执行。所以从A的角度看，V1，V2值是1，V3的值是2。

## 长事务
使用set autocommit=1，通过显示语句的方式启动事务。
对于一个需要频繁使用事务的业务，可以执行`commit work and chain`，提交事务并自动启动下一个事务。

```mysql
# 查询持续时间超过60s的长事务
select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(), trx_started)) > 60;
```

# MySQL索引
## 索引常见数据模型
### 哈希表
![mysql哈希表](https://ws4.sinaimg.cn/large/006tNbRwgy1fxkmimlk5fj30vq0ns75z.jpg)
哈希表适用于只有等值无序查询的场景，比如Memcached及其他一些NoSQL引擎。

### 有序数组
![mysql有序数组](https://ws3.sinaimg.cn/large/006tNbRwgy1fxkmk1a7sxj30vq0lvdh7.jpg)
可以通过二分法快速查找，查询效率高，但更新成本太高，只适用于静态存储引擎。比如存2017年的数据这样的不会再修改的数据。

### 搜索树
![mysql搜索树](https://ws4.sinaimg.cn/large/006tNbRwgy1fxkmoj6po0j30vq0ns0ux.jpg)
N叉树由于在读写上的性能优点，以及适配磁盘的访问模式，已经被广泛应用在数据库引擎中了。

## 索引类型
索引类型分为主键索引（聚簇索引）和非主键索引。
主键索引的叶子节点存放的是整行数据。
非主键索引的叶子节点存放的是主键的值。

**主键长度越小，普通索引的叶子节点就越小，普通索引占用的空间也就越小。**



### 索引重建
索引可能因为删除，或者页分裂等原因，导致数据页有空洞，重建索引的过程会创建一个新的索引，把数据按顺序插入，这样页面的利用率最高，也就是索引更紧凑、更省空间。

*重建主键索引不需要通过删除主键再创建主键的方式，因为不论删除主键还是创建主键，都会将整个表重建，所以连着执行这个两个语句，第一个删除主键的操作就白做了。可以通过这个语句替换：**`alter table T engine=InnoDB`**。*