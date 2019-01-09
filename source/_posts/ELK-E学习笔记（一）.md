---
title: ELK-E学习笔记（一）
date: 2019-01-09 14:18:11
author: ACE NI
tags:
  - ELK
categories:
  - 高并发架构
---

# 基础入门

一个 Elasticsearch 请求和任何 HTTP 请求一样由若干相同的部件组成：

```
curl -X<VERB> '<PROTOCOL>://<HOST>:<PORT>/<PATH>?<QUERY_STRING>' -d '<BODY>'
```

被 `< >` 标记的部件：

| 参数           | 说明                                                         |
| :------------- | :----------------------------------------------------------- |
| `VERB`         | 适当的 HTTP *方法* 或 *谓词* : `GET`、 `POST`、 `PUT`、 `HEAD` 或者 `DELETE`。 |
| `PROTOCOL`     | `http` 或者 `https`（如果你在 Elasticsearch 前面有一个 `https` 代理） |
| `HOST`         | Elasticsearch 集群中任意节点的主机名，或者用 `localhost` 代表本地机器上的节点。 |
| `PORT`         | 运行 Elasticsearch HTTP 服务的端口号，默认是 `9200` 。       |
| `PATH`         | API 的终端路径（例如 `_count` 将返回集群中文档数量）。Path 可能包含多个组件，例如：`_cluster/stats` 和 `_nodes/stats/jvm` 。 |
| `QUERY_STRING` | 任意可选的查询字符串参数 (例如 `?pretty` 将格式化地输出 JSON 返回值，使其更容易阅读) |
| `BODY`         | 一个 JSON 格式的请求体 (如果请求需要的话)                    |

<!--more-->

`POST`  /index/type    创建新文档。

`POST` /index/type/_id  部分更新一个文档。

`PUT` /index/type/_id   创建一个新文档或者替换一个现有的文档。

`DELETE` /index/type_id  删除一个文档。

`mget`   取回多个文档 

https://www.elastic.co/guide/cn/elasticsearch/guide/current/_Retrieving_Multiple_Documents.html

`bulk`  批量操作

https://www.elastic.co/guide/cn/elasticsearch/guide/current/bulk.html

## 底层原理

查找一个文档到一个分片中

`shard = hash(routing) % number_of_primary_shards`

`routing` 是一个可变值，默认是文档的 `_id` ，也可以设置成一个自定义的值。

`routing` 通过 hash 函数生成一个数字，然后这个数字再除以 `number_of_primary_shards` （主分片的数量）后得到 **余数** 。这个分布在 `0` 到 `number_of_primary_shards-1` 之间的余数，就是我们所寻求的文档所在分片的位置。

`consistency`

consistency，即一致性。在默认设置下，即使仅仅是在试图执行一个_写_操作之前，主分片都会要求必须要有 _规定数量(quorum)_（或者换种说法，也即必须要有大多数）的分片副本处于活跃可用状态，才会去执行_写_操作(其中分片副本可以是主分片或者副本分片)。这是为了避免在发生网络分区故障（network partition）的时候进行_写_操作，进而导致数据不一致。_规定数量_即：

`int( (primary + number_of_replicas) / 2 ) + 1`

`consistency` 参数的值可以设为 `one` （只要主分片状态 ok 就允许执行_写_操作）,`all`（必须要主分片和所有副本分片的状态没问题才允许执行_写_操作）, 或 `quorum` 。默认值为 `quorum` , 即大多数的分片副本状态没问题就允许执行_写_操作。

注意，*规定数量* 的计算公式中 `number_of_replicas` 指的是在索引设置中的设定副本分片数，而不是指当前处理活动状态的副本分片数。如果你的索引设置中指定了当前索引拥有三个副本分片，那规定数量的计算结果即：

`int( (primary + 3 replicas) / 2 ) + 1 = 3`

如果此时你只启动两个节点，那么处于活跃状态的分片副本数量就达不到规定数量，也因此您将无法索引和删除任何文档。

`timeout`

如果没有足够的副本分片会发生什么？ Elasticsearch会等待，希望更多的分片出现。默认情况下，它最多等待1分钟。 如果你需要，你可以使用 `timeout` 参数 使它更早终止： `100` 100毫秒，`30s` 是30秒。



## 深入搜索

一定要了解 `term` 和 `terms` 是 *包含（contains）* 操作，而非 *等值（equals）* （判断）。

https://www.elastic.co/guide/cn/elasticsearch/guide/cn/_finding_multiple_exact_values.html



`range` 查询同样可以处理字符串字段， 字符串范围可采用 *字典顺序（lexicographically）* 或字母顺序（alphabetically）。例如，下面这些字符串是采用字典序（lexicographically）排序的：

- 5, 50, 6, B, C, a, ab, abb, abc, b

数字和日期字段的索引方式使高效地范围计算成为可能。 但字符串却并非如此，要想对其使用范围过滤，Elasticsearch 实际上是在为范围内的每个词项都执行 `term` 过滤器，这会比日期或数字的范围过滤慢许多。

字符串范围在过滤 *低基数（low cardinality）* 字段（即只有少量唯一词项）时可以正常工作，但是唯一词项越多，字符串范围的计算会越慢。



## 全文搜索









