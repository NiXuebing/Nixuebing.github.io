---
title: MySQL学习笔记（二）
date: 2018-11-25 21:49:52
tags:
 - mysql
categories:
 - 高并发架构
---
# 索引相关概念
## 回表
```mysql
mysql> create table T ( 
ID int primary key,
k int NOT NULL DEFAULT 0,
s varchar(16) NOT NULL DEFAULT '',
index k(k))
engine=InnoDB;

insert into T values(100, 1, 'aa'),(200, 2, 'bb'),(300, 3, 'cc'),(500, 5, 'ee'),(600, 6, 'ff'),(700, 7, 'gg');
```
执行语句`select * from T where k between 3 and 5`

![mysql回表](https://ws3.sinaimg.cn/large/006tNbRwgy1fxknsizf2ej30vq0nsgn3.jpg)
1. 在k索引树上找到`k=3`的记录，取得`ID=300`；
2. 再到ID索引树查到`ID=300`对应的R3；
3. 在k索引树取下一个值`k=5`，取得`ID=500`；
4. 再回到ID索引树查到`ID=500`对应的R4；
5. 在k索引树取下一个值`k=6`，不满足条件，循环结束。

这个查询过程读取了k索引树的3条记录(步骤1、3、5)，回表了两次(步骤2、4)。

## 覆盖索引
如果执行的语句是`select ID from T where k between 3 and 5`，ID为主键已经在k索引树上了，因此可以直接提供查询结果，不需要回表。

**由于覆盖索引可以减少树的搜索次数，可以显著提升查询性能。**

## 最左前缀原则

![mysql最左原则](https://ws2.sinaimg.cn/large/006tNbRwgy1fxko9pise5j30vq0mqwg5.jpg)
用(name, age)这个联合索引分析
如果要查的是所有名字第一个字是“张”的人，执行的SQL语句的条件是`where name like '张%'`，也能够用上这个索引。

可以看到，不只是索引的全部定义，只要满足最左前缀，就可以利用索引来加速检索。这个最左前缀可以是联合索引的最左N个字段，也可以是字符串索引的最左M个字符。

**第一原则是，如果通过调整顺序，可以少维护一个索引，那么这个顺序往往就是需要优先考虑采用的。**

## 索引下推
``select * from tuser where name like '张%' and age=10 and ismale=1;`
通过最左前缀原则，过滤出“张”开头的索引，再判断`age=10`再过滤一遍再做回表操作。

MySQL 5.6后引入了索引下推优化，可以在索引遍历过程中，对索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数。

