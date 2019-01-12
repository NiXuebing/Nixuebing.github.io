---

title: ELK-E学习笔记（三）
date: 2019-01-10 16:41:51
author: ACE NI
tags:
  - ELK
categories:
  - 高并发架构
---

# 索引

禁止自动创建索引，你 可以通过在 `config/elasticsearch.yml` 的每个节点下添加下面的配置：

```
action.auto_create_index: false
```

使删除只限于特定名称指向的数据, 而不允许通过指定 `_all` 或通配符来删除指定索引库。

```
action.destructive_requires_name: true
```

<!-- more -->

## 索引设置

`number_of_shards`

每个索引的主分片数，默认值是 `5` 。这个配置在索引创建后不能修改。

`number_of_replicas`

每个主分片的副本数，默认值是 `1` 。对于活动的索引库，这个配置可以随时修改。

自定义分析器

https://www.elastic.co/guide/cn/elasticsearch/guide/current/custom-analyzers.html

## 根对象

### 元数据：`_source`字段

这个字段的存储几乎总是我们想要的，因为它意味着下面的这些：

- 搜索结果包括了整个可用的文档——不需要额外的从另一个的数据仓库来取文档。
- 如果没有 `_source` 字段，部分 `update` 请求不会生效。
- 当你的映射改变时，你需要重新索引你的数据，有了_source字段你可以直接从Elasticsearch这样做，而不必从另一个（通常是速度更慢的）数据仓库取回你的所有文档。
- 当你不需要看到整个文档时，单个字段可以从 `_source` 字段提取和通过 `get` 或者 `search` 请求返回。
- 调试查询语句更加简单，因为你可以直接看到每个文档包括什么，而不是从一列id猜测它们的内容。

### 元数据：`_all`字段

一个把其它字段值当作一个大字符串来索引的特殊字段。 

通过 `include_in_all` 设置来逐个控制字段是否要包含在 `_all` 字段中，默认值是 `true`。在一个对象(或根对象)上设置 `include_in_all` 可以修改这个对象中的所有字段的默认行为。

你可能想要保留 `_all` 字段作为一个只包含某些特定字段的全文字段，例如只包含 `title`，`overview`，`summary` 和 `tags`。 相对于完全禁用 `_all` 字段，你可以为所有字段默认禁用 `include_in_all` 选项，仅在你选择的字段上启用。

### 元数据：文档标识

文档标识与四个元数据字段 相关：

- `_id`

  文档的 ID 字符串

- `_type`

  文档的类型名

- `_index`

  文档所在的索引

- `_uid`

  `_type` 和 `_id` 连接在一起构造成 `type#id`

默认情况下， `_uid` 字段是被存储（可取回）和索引（可搜索）的。 `_type` 字段被索引但是没有存储，`_id` 和 `_index` 字段则既没有被索引也没有被存储，这意味着它们并不是真实存在的。

尽管如此，你仍然可以像真实字段一样查询 `_id` 字段。Elasticsearch 使用 `_uid` 字段来派生出 `_id` 。

## 重新索引

```js
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter"
  }
}
```

## 索引别名

```js
POST /_aliases
{
    "actions": [
        { "remove": { "index": "my_index_v1", "alias": "my_index" }},
        { "add":    { "index": "my_index_v2", "alias": "my_index" }}
    ]
}
```

# 分片

**新的文档被添加到内存缓冲区并且被追加到了事务日志**

![mark](http://pic-cloud.ice-leaf.top/pic-cloud/20190112/z3AJiIOXriPF.png?imageslim)

 **刷新（refresh）完成后, 缓存被清空但是事务日志不会**

![mark](http://pic-cloud.ice-leaf.top/pic-cloud/20190112/ig35kVGz0oA2.png?imageslim)

- 这些在内存缓冲区的文档被写入到一个新的段中，且没有进行 `fsync` 操作。
- 这个段被打开，使其可被搜索。
- 内存缓冲区被清空。

**事务日志不断积累文档**

![mark](http://pic-cloud.ice-leaf.top/pic-cloud/20190112/fLC5y2zhHhwq.png?imageslim)

**在刷新（flush）之后，段被全量提交，并且事务日志被清空**

![mark](http://pic-cloud.ice-leaf.top/pic-cloud/20190112/eVHJw2E4vQw9.png?imageslim)

每隔一段时间--例如 translog 变得越来越大--索引被刷新（flush）；一个新的 translog 被创建，并且一个全量提交被执行

- 所有在内存缓冲区的文档都被写入一个新的段。
- 缓冲区被清空。
- 一个提交点被写入硬盘。
- 文件系统缓存通过 `fsync` 被刷新（flush）。
- 老的 translog 被删除。

translog 提供所有还没有被刷到磁盘的操作的一个持久化纪录。当 Elasticsearch 启动的时候， 它会从磁盘中使用最后一个提交点去恢复已知的段，并且会重放 translog 中所有在最后一次提交后发生的变更操作。

translog 也被用来提供实时 CRUD 。当你试着通过ID查询、更新、删除一个文档，它会在尝试从相应的段中检索之前， 首先检查 translog 任何最近的变更。这意味着它总是能够实时地获取到文档的最新版本。

