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

## 底层原理

一个运行中的 Elasticsearch 实例称为一个 节点，而集群是由一个或者多个拥有相同 `cluster.name` 配置的节点组成， 它们共同承担数据和负载的压力。当有节点加入集群中或者从集群中移除节点时，集群将会**重新平均分布**所有的数据。

### 索引

保存相关数据的地方。 索引实际上是指向一个或者多个物理 *分片* 的 *逻辑命名空间* 。

### 分片

一个底层的 *工作单元* ，它仅保存了 全部数据中的一部分。一个分片是一个 Lucene 的实例，以及它本身就是一个完整的搜索引擎。

一个分片可以是 *主* 分片或者 *副本* 分片。 索引内任意一个文档都归属于一个主分片，所以主分片的数目决定着索引能够保存的最大数据量。

一个主分片最大能够存储 Integer.MAX_VALUE - 128 （2^31 -129） 个文档，但是实际最大值还需要参考你的使用场景：包括你使用的硬件， 文档的大小和复杂程度，索引和查询文档的方式以及你期望的响应时长。

主分片的数目在索引创建时 就已经确定了下来。实际上，这个数目定义了这个索引能够 *存储* 的最大数据量。（实际大小取决于你的数据、硬件和使用场景。） 但是，读操作——搜索和返回数据——可以同时被主分片 *或* 副本分片所处理，所以当你拥有越多的副本分片时，也将拥有越高的吞吐量。

### 文档

一个文档的 `_index` 、 `_type` 和 `_id` 唯一标识一个文档。

文档 API：  `get` 、 `index` 、 `delete` 、 `bulk` 、 `update` 以及 `mget` 

#### 路由一个文档到一个分片中

`shard = hash(routing) % number_of_primary_shards`

`routing` 是一个可变值，默认是文档的 `_id` ，也可以设置成一个自定义的值。

`routing` 通过 hash 函数生成一个数字，然后这个数字再除以 `number_of_primary_shards` （主分片的数量）后得到 **余数** 。这个分布在 `0` 到 `number_of_primary_shards-1` 之间的余数，就是我们所寻求的文档所在分片的位置。

#### 新建、索引和删除文档

`POST`  /index/type    创建新文档。

`PUT` /index/type/_id   创建一个新文档或者替换一个现有的文档。

`DELETE` /index/type_id  删除一个文档。

![mark](http://pic-cloud.ice-leaf.top/pic-cloud/20190110/dHa6TH7tkTjk.png?imageslim)

1. 客户端向 `Node 1` 发送新建、索引或者删除请求。
2. 节点使用文档的 `_id` 确定文档属于分片 0 。请求会被转发到 `Node 3`，因为分片 0 的主分片目前被分配在 `Node 3` 上。
3. `Node 3` 在主分片上面执行请求。如果成功了，它将请求并行转发到 `Node 1` 和 `Node 2` 的副本分片上。一旦所有的副本分片都报告成功, `Node 3` 将向协调节点报告成功，协调节点向客户端报告成功。

在客户端收到成功响应时，文档变更已经在主分片和所有副本分片执行完成，变更是安全的。

有一些可选的请求参数允许您影响这个过程，可能以数据安全为代价提升性能。这些选项很少使用，因为Elasticsearch已经很快

`consistency`

consistency，即一致性。在默认设置下，即使仅仅是在试图执行一个_写_操作之前，主分片都会要求必须要有 _规定数量(quorum)_（或者换种说法，也即必须要有大多数）的分片副本处于活跃可用状态，才会去执行_写_操作(其中分片副本可以是主分片或者副本分片)。这是为了避免在发生网络分区故障（network partition）的时候进行_写_操作，进而导致数据不一致。_规定数量_即：

`int( (primary + number_of_replicas) / 2 ) + 1`

`consistency` 参数的值可以设为 `one` （只要主分片状态 ok 就允许执行_写_操作）,`all`（必须要主分片和所有副本分片的状态没问题才允许执行_写_操作）, 或 `quorum` 。默认值为 `quorum` , 即大多数的分片副本状态没问题就允许执行_写_操作。

注意，*规定数量* 的计算公式中 `number_of_replicas` 指的是在索引设置中的设定副本分片数，而不是指当前处理活动状态的副本分片数。如果你的索引设置中指定了当前索引拥有三个副本分片，那规定数量的计算结果即：

`int( (primary + 3 replicas) / 2 ) + 1 = 3`

如果此时你只启动两个节点，那么处于活跃状态的分片副本数量就达不到规定数量，也因此您将无法索引和删除任何文档。

`timeout`

如果没有足够的副本分片会发生什么？ Elasticsearch会等待，希望更多的分片出现。默认情况下，它最多等待1分钟。 如果你需要，你可以使用 `timeout` 参数 使它更早终止： `100` 100毫秒，`30s` 是30秒。

#### 取回一个文档

取回一个文档： `GET _indx/_type/_id`

取回文档的一部分：`GET _index/_type/_id/_source=_field`

![mark](http://pic-cloud.ice-leaf.top/pic-cloud/20190110/PUBfrhKT2vEc.png?imageslim)

1. 客户端向 `Node 1` 发送获取请求。

2. 节点使用文档的 `_id` 来确定文档属于分片 `0` 。分片 `0` 的副本分片存在于所有的三个节点上。 在这种情况下，它将请求转发到 `Node 2` 。

3. `Node 2` 将文档返回给 `Node 1` ，然后将文档返回给客户端。

在处理读取请求时，协调结点在每次请求的时候都会通过轮询所有的副本分片来达到负载均衡。

#### 局部更新文档

`POST` /index/type/_id  部分更新一个文档。

![mark](http://pic-cloud.ice-leaf.top/pic-cloud/20190110/7dcm0b6yIUIs.png?imageslim)

1. 客户端向 `Node 1` 发送更新请求。
2. 它将请求转发到主分片所在的 `Node 3` 。
3. `Node 3` 从主分片检索文档，修改 `_source` 字段中的 JSON ，并且尝试重新索引主分片的文档。 如果文档已经被另一个进程修改，它会重试步骤 3 ，超过 `retry_on_conflict` 次后放弃。
4. 如果 `Node 3` 成功地更新文档，它将新版本的文档并行转发到 `Node 1` 和 `Node 2` 上的副本分片，重新建立索引。 一旦所有副本分片都返回成功， `Node 3` 向协调节点也返回成功，协调节点向客户端返回成功。

**当主分片把更改转发到副本分片时， 它不会转发更新请求。 相反，它转发完整文档的新版本。**

在 Elasticsearch 中文档是 *不可改变* 的，不能修改它们。 相反，如果想要更新现有的文档，需要 *重建索引*或者进行替换。

调用`POST /index/type/_id`做部分更新的好处在于这个过程发生在分片内部，这样就避免了多次请求的网络开销。通过减少检索和重建索引步骤之间的时间，我们也减少了其他进程的变更带来冲突的可能性。

*在使用`spring-boot-starter-data-elasticsearch`时，由于API都被封装了，不确定实际调用的是`index API`还是`update API`，有待具体确认。*

`mget`   取回多个文档 

![mark](http://pic-cloud.ice-leaf.top/pic-cloud/20190110/VrEPrtvYxiBV.png?imageslim)

1. 客户端向 `Node 1` 发送 `mget` 请求。
2. `Node 1` 为每个分片构建多文档获取请求，然后并行转发这些请求到托管在每个所需的主分片或者副本分片的节点上。一旦收到所有答复， `Node 1` 构建响应并将其返回给客户端。

https://www.elastic.co/guide/cn/elasticsearch/guide/current/_Retrieving_Multiple_Documents.html

`bulk`  批量操作

![mark](http://pic-cloud.ice-leaf.top/pic-cloud/20190110/GuE5crTPJmwV.png?imageslim)

1. 客户端向 `Node 1` 发送 `bulk` 请求。
2. `Node 1` 为每个节点创建一个批量请求，并将这些请求并行转发到每个包含主分片的节点主机。
3. 主分片一个接一个按顺序执行每个操作。当每个操作成功时，主分片并行转发新文档（或删除）到副本分片，然后执行下一个操作。 一旦所有的副本分片报告所有操作成功，该节点将向协调节点报告成功，协调节点将这些响应收集整理并返回给客户端。

整个批量请求都需要由接收到请求的节点加载到内存中，因此该请求越大，其他请求所能获得的内存就越少。 批量请求的大小有一个最佳值，大于这个值，性能将不再提升，甚至会下降。 但是最佳值不是一个固定的值。它完全取决于硬件、文档的大小和复杂度、索引和搜索的负载的整体情况。

幸运的是，很容易找到这个 *最佳点* ：通过批量索引典型文档，并不断增加批量大小进行尝试。 当性能开始下降，那么你的批量大小就太大了。一个好的办法是开始时将 1,000 到 5,000 个文档作为一个批次, 如果你的文档非常大，那么就减少批量的文档个数。

密切关注你的批量请求的物理大小往往非常有用，一千个 1KB 的文档是完全不同于一千个 1MB 文档所占的物理大小。 一个好的批量大小在开始处理后所占用的物理大小约为 5-15 MB。

https://www.elastic.co/guide/cn/elasticsearch/guide/current/bulk.html

`并发冲突` 乐观并发控制

https://www.elastic.co/guide/cn/elasticsearch/guide/current/optimistic-concurrency-control.html









