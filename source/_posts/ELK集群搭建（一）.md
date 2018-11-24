title: ELK集群搭建（一）
author: ACE NI
tags:
  - ELK
categories:
  - 高并发架构
date: 2018-04-13 20:32:00
---
# ELK简介

ELK Stack 是软件集合 Elasticsearch、Logstash、Kibana 的简称，由这三个软件及其相关的组件可以打造大规模日志实时处理系统。

<!--more-->

**E - Elasticsearch**

Elasticsearch 是基于 JSON 的分布式搜索和分析引擎，专为实现水平扩展、高可用和管理便捷性而设计。

**L - Logstash**

Logstash 是开源的服务器端数据处理管道，能够同时 从多个来源采集数据、转换数据，然后将数据发送到您最喜欢的 “存储库” 中。

**K - Kibana**

Kibana 能够以图表的形式呈现数据，并且具有可扩展的用户界面，供您全方位配置和管理 Elastic Stack。


**ELK使用场景**

- 日志分析 -- 快速、可扩展的日志记录分析
- 网站搜索 -- 创建良好的搜索体验
- 安全分析 -- 快速且规模化的交互式调查
- APM -- 深入了解应用程序的性能
- 应用搜索 -- 搜索文档、地理数据等

[ELK产品介绍](https://www.elastic.co/cn/products)

---

# 搭建准备

- CentOS7-mq01: 172.18.1.152
- CentOS7-mq02: 172.18.1.153
- CentOS7-mq03: 172.18.1.154
- CentOS7-kibana: 172.18.10.106
- [elasticsearch-6.2.3.tar.gz](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.2.3.tar.gz)
- [logstash-6.2.3.tar.gz](https://artifacts.elastic.co/downloads/logstash/logstash-6.2.3.tar.gz)
- [kibana-6.2.3-linux-x86_64.tar.gz](https://artifacts.elastic.co/downloads/kibana/kibana-6.2.3-linux-x86_64.tar.gz)

创建用户 elk
```
useradd elk
passwd elk
```

---

# 集群搭建

## Elasticsearch
```bash
[mq01]$ pwd
/data/share/download

[mq01]$ wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.2.3.tar.gz
[mq01]$ tar zxvf elasticsearch-6.2.3.tar.gz -C /usr/local/server/
[mq01]$ cd /usr/local/server/

# 更改所属用户
[mq01]$ chown -R elk:elk elasticsearch-6.2.3/
[mq01]$ cd elasticsearch-6.2.3/
[mq01]$ ll
drwxr-xr-x  3 elk elk   4096 4月  14 00:37 bin
drwxr-xr-x  3 elk elk   4096 4月  14 00:37 config
drwxr-xr-x  2 elk elk   4096 3月  13 18:08 lib
-rw-r--r--  1 elk elk  11358 3月  13 18:02 LICENSE.txt
drwxr-xr-x  2 elk elk   4096 4月  14 00:43 logs
drwxr-xr-x 16 elk elk   4096 3月  13 18:08 modules
-rw-r--r--  1 elk elk 191887 3月  13 18:07 NOTICE.txt
drwxr-xr-x  3 elk elk   4096 4月  14 00:37 plugins
-rw-r--r--  1 elk elk   9268 3月  13 18:02 README.textile

[mq01]$ cd config/
[mq01]$ ll
-rw-rw---- 1 elk elk 2938 4月  14 00:32 elasticsearch.yml
-rw-rw---- 1 elk elk 2767 3月  13 18:02 jvm.options
-rw-rw---- 1 elk elk 5091 3月  13 18:02 log4j2.properties

[mq01]$vi elasticsearch.yml
cluster.name: dev-elasticsearch
node.name: node-1
node.attr.rack: r1
path.data: /data/share/server_data/mq_01/elasticsearch
path.logs: /data/share/server_log/mq_01/elasticsearch
network.host: 172.18.1.152
http.port: 9200
discovery.zen.ping.unicast.hosts: ["172.18.1.152", "172.18.1.153", "172.18.1.154"]
discovery.zen.minimum_master_nodes: 2
gateway.recover_after_nodes: 3

# 安装X-Pack for Elasticsearch
[mq01]$ cd ../bin
[mq01]$ ./elasticsearch-plugin install x-pack

# 切换为elk用户，以后台进程启动
[mq01]$ chown -R elk:elk /data/share/server_data/mq_01/elasticsearch
[mq01]$ chown -R elk:elk /data/share/server_log/mq_01/elasticsearch
[mq01]$ su elk
[mq01]$ nohup ./elasticsearch > ../logs/run.log 2>&1 &

# 设置密码
[mq01]$ ./x-pack/setup-passwords interactive
```
mq02、mq03相同的安装步骤，文件路径和日志输出路径做相应修改

[Elasticsearch集群搭建](https://www.kancloud.cn/hanxt/elk/216151)

[Elasticsearch官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/install-elasticsearch.html)

[X-pack for Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/installing-xpack-es.html)

[异常处理](https://segmentfault.com/a/1190000011899522)


## logstash

```bash
[mq01]$ pwd
/data/share/download
[mq01]$ wget https://artifacts.elastic.co/downloads/logstash/logstash-6.2.3.tar.gz
[mq01]$ tar zxvf logstash-6.2.3.tar.gz -C /usr/local/server/
[mq01]$ cd /usr/local/server/

# 更改所属用户
[mq01]$ chown -R elk:elk logstash-6.2.3/
[mq01]$ cd logstash-6.2.3/
[mq01]$ ll
drwxr-xr-x 2 elk elk  4096 4月  13 22:21 bin
drwxr-xr-x 2 elk elk  4096 4月  13 23:47 config
-rw-r--r-- 1 elk elk  2276 3月  13 19:49 CONTRIBUTORS
drwxr-xr-x 2 elk elk  4096 3月  13 19:49 data
-rw-r--r-- 1 elk elk  3869 3月  13 19:53 Gemfile
-rw-r--r-- 1 elk elk 21170 3月  13 19:51 Gemfile.lock
drwxr-xr-x 6 elk elk  4096 4月  13 22:21 lib
-rw-r--r-- 1 elk elk   589 3月  13 19:49 LICENSE
drwxr-xr-x 4 elk elk  4096 4月  13 22:21 logstash-core
drwxr-xr-x 3 elk elk  4096 4月  13 22:21 logstash-core-plugin-api
drwxr-xr-x 4 elk elk  4096 4月  13 22:21 modules
-rw-rw-r-- 1 elk elk 28122 3月  13 19:53 NOTICE.TXT
drwxr-xr-x 3 elk elk  4096 4月  13 22:21 tools
drwxr-xr-x 4 elk elk  4096 4月  13 22:21 vendor

[mq01]$ cd config/
[mq01]$ ll
-rw-r--r-- 1 elk elk 1894 4月  13 23:27 jvm.options
-rw-r--r-- 1 elk elk 4466 3月  13 19:49 log4j2.properties
-rw-r--r-- 1 elk elk 6381 4月  13 23:39 logstash.yml
-rw-r--r-- 1 elk elk 3244 3月  13 19:49 pipelines.yml
-rw-r--r-- 1 elk elk 1802 4月  13 23:47 startup.options

# 安装X-Pack for Logstash
[mq01]$ cd ../bin
[mq01]$ ./logstash-plugin install x-pack


[mq01]$ vi logstash.yml
path.data: /data/share/server_data/mq_01/logstash
path.logs: /data/share/server_log/mq_01/logstash
# 数量等于CPU数
pipeline.workers: 4
# 数据落盘
queue.type: persisted

[mq01]$ vi startup.options
JAVACMD=/usr/local/server/jdk1.8.0_151/bin/java
LS_HOME=/usr/local/server/logstash-6.2.3
LS_USER=elk
LS_GROUP=elk
LS_PIDFILE=/data/share/server_log/mq_01/logstash/logstash.pid
LS_GC_LOG_FILE=/data/share/server_log/mq_01/logstash/gc.log

[mq01]$ chown -R elk:elk /data/share/server_data/mq_01/logstash
[mq01]$ chown -R elk:elk /data/share/server_log/mq_01/logstash

# 测试logstash
[mq01]$ ../bin/logstash -e 'input { stdin { } } output { stdout {} }'

```
mq02、mq03相同的安装步骤，文件路径和日志输出路径做相应修改

[ELK中文文档](https://www.kancloud.cn/hanxt/elk/153879)
[logstash官方文档](https://www.elastic.co/guide/en/logstash/6.2/installing-logstash.html)
