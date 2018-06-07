---
layout: post
categories: ELK
title: ELK elasticsearch 管理优化杂记
date: 2017-01-23 13:34:44 +0800
description: elasticsearch
keywords: ELK elasticsearch
---


## 索引分片： 从策略层面，控制分片分配的选择
>磁盘限额 为了保护节点数据安全，ES 会定时(cluster.info.update.interval，默认 30 秒)检查一下各节点的数据目录磁盘使用情况。
>在达到 cluster.routing.allocation.disk.watermark.low (默认 85%)的时候，新索引分片就不会再分配到这个节点上了。
>在达到 cluster.routing.allocation.disk.watermark.high (默认 90%)的时候，就会触发该节点现存分片的数据均衡，把数据挪到其他节点上去。
>这两个值不但可以写百分比，还可以写具体的字节,有些公司可能出于成本考虑，对磁盘使用率有一定的要求，需要适当抬高这个配置：

``` sh
# curl -XPUT localhost:9200/_cluster/settings -d '{
    "transient" : {
        "cluster.routing.allocation.disk.watermark.low" : "85%",
        "cluster.routing.allocation.disk.watermark.high" : "10gb",
        "cluster.info.update.interval" : "1m"
    }
}'
```

>热索引分片不均 默认情况下，ES 集群的数据均衡策略是以各节点的分片总数(indices_all_active)作为基准的。这对于搜索服务来说无疑是均衡搜索压力提高性能的好办法。但是对于 Elastic Stack 场景，一般压力集中在新索引的数据写入方面。正常运行的时候，也没有问题。但是当集群扩容时，新加入集群的节点，分片总数远远低于其他节点。这时候如果有新索引创建，ES 的默认策略会导致新索引的所有主分片几乎全分配在这台新节点上。整个集群的写入压力，压在一个节点上，结果很可能是这个节点直接被压死，集群出现异常。 所以，对于 Elastic Stack 场景，强烈建议大家预先计算好索引的分片数后，配置好单节点分片的限额。比如，一个 5 节点的集群，索引主分片 10 个，副本 1 份。则平均下来每个节点应该有 4 个分片，那么就配置：

``` bash
# curl -s -XPUT http://127.0.0.1:9200/logstash-2015.05.08/_settings -d '{
    "index": { "routing.allocation.total_shards_per_node" : "5" }
}'
```

注意，这里配置的是 5 而不是 4。因为我们需要预防有机器故障，分片发生迁移的情况。如果写的是 4，那么分片迁移会失败。
此外，另一种方式则更加玄妙，Elasticsearch 中有一系列参数，相互影响，最终联合决定分片分配：
>cluster.routing.allocation.balance.shard 节点上分配分片的权重，默认为 0.45。数值越大越倾向于在节点层面均衡分片。
>cluster.routing.allocation.balance.index 每个索引往单个节点上分配分片的权重，默认为 0.55。数值越大越倾向于在索引层面均衡分片。
>cluster.routing.allocation.balance.threshold 大于阈值则触发均衡操作。默认为1。


## reroute 接口
* reroute 接口支持五种指令：allocate_replica, allocate_stale_primary, allocate_empty_primary，move 和 cancel。
* 常用的一般是 allocate 和 move：
allocate_* 指令
* 因为负载过高等原因，有时候个别分片可能长期处于 UNASSIGNED 状态，我们就可以手动分配分片到指定节点上。默认情况下只允许手动分配副本分片(即使用 allocate_replica)，所以如果要分配主分片，需要单独加一个 accept_data_loss 选项：

``` sh
# curl -XPOST 127.0.0.1:9200/_cluster/reroute -d '{
  "commands" : [ {
        "allocate_stale_primary" :
            {
              "index" : "logstash-2015.05.27", "shard" : 61, "node" : "10.19.0.77", "accept_data_loss" : true
            }
        }
  ]
}'
```

* 因为负载过高，磁盘利用率过高，服务器下线，更换磁盘等原因，可以会需要从节点上移走部分分片：

``` sh
curl -XPOST 127.0.0.1:9200/_cluster/reroute -d '{
  "commands" : [ {
        "move" :
            {
              "index" : "logstash-2015.05.22", "shard" : 0, "from_node" : "10.19.0.81", "to_node" : "10.19.0.104"
            }
        }
  ]
}'
```

## 节点下线

* 集群中个别节点出现故障预警等情况，需要下线，也是 Elasticsearch 运维工作中常见的情况。如果已经稳定运行过一段时间的集群，每个节点上都会保存有数量不少的分片。这种时候通过 reroute 接口手动转移，就显得太过麻烦了。这个时候，有另一种方式：

``` sh
curl -XPUT 127.0.0.1:9200/_cluster/settings -d '{
  "transient" :{
      "cluster.routing.allocation.exclude._ip" : "10.0.0.1"
   }
}'
```
>Elasticsearch 集群就会自动把这个 IP 上的所有分片，都自动转移到其他节点上。等到转移完成，这个空节点就可以毫无影响的下线了。和 _ip 类似的参数还有 _host, _name 等。此外，这类参数不单是 cluster 级别，也可以是 index 级别。下一小节就是 index 级别的用例。


## 冷热数据的读写分离

Elasticsearch 集群一个比较突出的问题是: 用户做一次大的查询的时候, 非常大量的读 IO 以及聚合计算导致机器 Load 升高, CPU 使用率上升, 会影响阻塞到新数据的写入, 这个过程甚至会持续几分钟。所以，可能需要仿照 MySQL 集群一样，做读写分离。

>实施方案

N 台机器做热数据的存储, 上面只放当天的数据。这 N 台热数据节点上面的 elasticsearc.yml 中配置 

```
node.tag: hot
```

之前的数据放在另外的 M 台机器上。这 M 台冷数据节点中配置 node.tag: stale
模板中控制对新建索引添加 hot 标签：

``` json
{
 "order" : 0,
 "template" : "*",
 "settings" : {
   "index.routing.allocation.require.tag" : "hot"
 }
}
```

每天计划任务更新索引的配置, 将 tag 更改为 stale, 索引会自动迁移到 M 台冷数据节点

``` sh
# curl -XPUT http://127.0.0.1:9200/indexname/_settings -d'
{
"index": {
   "routing": {
      "allocation": {
         "require": {
            "tag": "stale"
         }
      }
  }
}
}
```

>这样，写操作集中在 N 台热数据节点上，大范围的读操作集中在 M 台冷数据节点上。避免了堵塞影响
* https://www.elastic.co/guide/en/elasticsearch/reference/master/shard-allocation-filtering.html
* http://kibana.logstash.es/content/elasticsearch/principle/shard-allocate.html

---    

## 集群自动发现
* fd 是 fault detection 
* 考虑到节点有时候因为高负载，慢 `GC` `[垃圾回收]` 等原因会偶尔没及时响应 ping ,一般建议稍加大 Fault Detection 的超时时间。
* discovery.zen.ping_timeout 仅在加入或者选举 master 主节点的时候才起作用；
* discovery.zen.fd.ping_timeout 在稳定运行的集群中，master检测所有节点，以及节点检测 master是否畅通时长期有用

``` sh
# 集群自动发现
discovery.zen.ping.unicast.hosts: ["es0","es1", "es2","es3","es4"]    
# 超时时间(根据实际情况调整)
discovery.zen.fd.ping_timeout: 120s
# 重试次数，防止GC[Garbage collection]节点不响应被剔除
discovery.zen.fd.ping_retries: 6
# 运行间隔
discovery.zen.fd.ping_interval: 30s                
```

## 慢查询配置
* es5.X 不支持配置文件设定
* 根据提示进行慢查询设定
* [Slow log](https://www.elastic.co/guide/en/elasticsearch/reference/5.2/index-modules-slowlog.html#index-slow-log)


``` sh
*************************************************************************************
Found index level settings on node level configuration.

Since elasticsearch 5.x index level settings can NOT be set on the nodes 
configuration like the elasticsearch.yaml, in system properties or command line 
arguments.In order to upgrade all indices the settings must be updated via the 
/${index}/_settings API. Unless all settings are dynamic all indices must be closed 
in order to apply the upgradeIndices created in the future should use index templates 
to set default values. 

Please ensure all required values are updated on all indices by executing: 

curl -XPUT 'http://localhost:9200/_all/_settings?preserve_existing=true' -d '{
  "index.indexing.slowlog.level" : "info",
  "index.indexing.slowlog.source" : "1000",
  "index.indexing.slowlog.threshold.index.debug" : "2s",
  "index.indexing.slowlog.threshold.index.info" : "5s",
  "index.indexing.slowlog.threshold.index.trace" : "500ms",
  "index.indexing.slowlog.threshold.index.warn" : "10s",
  "index.search.slowlog.threshold.fetch.debug" : "500ms",
  "index.search.slowlog.threshold.fetch.info" : "800ms",
  "index.search.slowlog.threshold.fetch.trace" : "200ms",
  "index.search.slowlog.threshold.fetch.warn" : "1s",
  "index.search.slowlog.threshold.query.debug" : "2s",
  "index.search.slowlog.threshold.query.info" : "5s",
  "index.search.slowlog.threshold.query.trace" : "500ms",
  "index.search.slowlog.threshold.query.warn" : "10s"
}'

```


## API 接口
### 数据写入

``` sh
# curl -XPOST http://127.0.0.1:9200/logstash-2015.06.21/testlog -d '{
    "date" : "1434966686000",
    "user" : "chenlin7",
    "mesg" : "first message into Elasticsearch"
}'
```
命令返回响应结果为：

``` sh
{"_index":"logstash-2015.06.21","_type":"testlog","_id":"AU4ew3h2nBE6n0qcyVJK","_version":1,"created":true}
```

### 数据读取

``` sh
# curl -XGET http://127.0.0.1:9200/logstash-2015.06.21/testlog/AU4ew3h2nBE6n0qcyVJK
# curl -XGET http://127.0.0.1:9200/logstash-2015.06.21/testlog/AU4ew3h2nBE6n0qcyVJK/_source 

GET apache-2016.12.30/testlog/AVlPEWLkYdfL4HTWelef/_source
GET apache-2016.12.30/testlog/AVlPEWLkYdfL4HTWelef?fields=user  # 指定字段
```

### 数据删除
``` sh
# curl -XDELETE http://127.0.0.1:9200/logstash-2015.06.21/testlog/AU4ew3h2nBE6n0qcyVJK
删除不单针对单条数据，还可以删除整个整个索引。甚至可以用通配符。
# curl -XDELETE http://127.0.0.1:9200/logstash-2015.06.0*

在 Elasticsearch 2.x 之前，可以通过查询语句删除，也可以删除某个 _type 内的数据。现在都已经不再内置支持，改为 Delete by Query 插件。因为这种方式本身对性能影响较大！
```

### 数据更新

``` sh
已经写过的数据，同样还是可以修改的。有两种办法，一种是全量提交，即指明 _id 再发送一次写入请求。
# curl -XPOST http://127.0.0.1:9200/logstash-2015.06.21/testlog/AU4ew3h2nBE6n0qcyVJK -d '{
    "date" : "1434966686000",
    "user" : "chenlin7",
    "mesg" " "first message into Elasticsearch but version 2"
}'

另一种是局部更新，使用 /_update 接口：
# curl -XPOST 'http://127.0.0.1:9200/logstash-2015.06.21/testlog/AU4ew3h2nBE6n0qcyVJK/_update' -d '{
    "doc" : {
        "user" : "someone"
    }
}'

或者
# curl -XPOST 'http://127.0.0.1:9200/logstash-2015.06.21/testlog/AU4ew3h2nBE6n0qcyVJK/_update' -d '{
    "script" : "ctx._source.user = \"someone\""
}'
```

### 分片存储设置
``` sh
PUT /_cluster/settings{
  "transient": {
    "cluster.routing.allocation.disk.watermark.low": "80%",
    "cluster.routing.allocation.disk.watermark.high": "5gb",
    "cluster.info.update.interval": "1m"
  }
}
```

## 搜索请求

``` sh
全文搜索
# curl -XGET http://127.0.0.1:9200/logstash-2015.06.21/testlog/_search?q=first
# curl -XGET http://127.0.0.1:9200/logstash-2015.06.21/testlog/_search?q=user:"chenlin7"
{
  "took": 2,   Elasticsearch 执行这个搜索的耗时，以毫秒为单位
  "timed_out": false,  搜索是否超时 
  "_shards": { 指出多少个分片被搜索了，同时也指出了成功/失败的被搜索的shards 的数量
    "total": 5,  
    "successful": 5,    
    "failed": 0     
  },
  "hits" : {   搜索结果
    "total" : 579,  匹配查询条件的文档的总数目
    "max_score" : 0.12633952,
    "hits" : [   真正的搜索结果数组（默认是前10个文档）
      {
        "_index" : "logstash-2015.06.21",
        "_type" : "logs",
        "_id" : "AVmvfgGaSF0Iy2LFN0EH",
        "_score" : 0.12633952,
        "_source" : {
          "user" : "chenlin7"
        }
      }
    ......


```

### querystring 语法

* 上例中，?q=后面写的，就是 querystring 语法。鉴于这部分内容会在 Kibana 上经常使用，这里详细解析一下语法：
* 全文检索：直接写搜索的单词，如上例中的 first；
* 单字段的全文检索：在搜索单词之前加上字段名和冒号，比如如果知道单词 first 肯定出现在 mesg 字段，可以写作 mesg:first；
* 单字段的精确检索：在搜索单词前后加双引号，比如 user:"chenlin7"；
* 多个检索条件的组合：可以使用 NOT, AND 和 OR 来组合检索，注意必须是大写。比如 user:("chenlin7" OR "chenlin") AND NOT mesg:first；
* 字段是否存在：_exists_:user 表示要求 user 字段存在，_missing_:user 表示要求 user 字段不存在；
* 通配符：用 ? 表示单字母，* 表示任意个字母。比如 fir?t mess*；
* 正则：需要比通配符更复杂一点的表达式，可以使用正则。比如 mesg:/mes{2}ages?/。注意 ES 中正则性能很差，而且支持的功能也不是特别强大，尽量不要使用。
* ES 支持的正则语法见：https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-regexp-query.html#regexp-syntax；
* 近似搜索：用 ~ 表示搜索单词可能有一两个字母写的不对，请 ES 按照相似度返回结果。比如 frist~；
* 范围搜索：对数值和时间，ES 都可以使用范围搜索，比如：rtt:>300，date:["now-6h" TO "now"} 等。其中，[] 表示端点数值包含在范围内，{} 表示端点数值不包含在范围内；
* 完整语法：
https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-queries.html

term query 的写法

``` sh
# curl -XGET http://127.0.0.1:9200/_search -d '
{
  "query": {
    "term": {
      "user": "chenlin7" 
    }
  }
}'

```

## 集群健康状态
```
$ curl -XGET 127.0.0.1:9200/_cluster/health?pretty
```

* green 绿灯，所有分片都正确运行，集群非常健康。
* yellow 黄灯，所有主分片都正确运行，但是有副本分片缺失。这种情况意味着 ES 当前还是正常运行的，但是有一定风险。注意，在 Kibana4 的 server 端启动逻辑中，即使是黄灯状态，Kibana 4 也会拒绝启动，死循环等待集群状态变成绿灯后才能继续运行。
* red 红灯，有`主分片缺失`。这部分数据完全不可用。而考虑到 ES 在写入端是简单的取余算法，轮到这个分片上的数据也会持续写入报错。

## level 请求参数
>接口请求的时候，可以附加一个 level 参数，指定输出信息以 indices 还是 shards 级别显示。当然，一般来说，indices 级别就够了。

* curl  -XGET http://127.0.0.1:9200/_cluster/health?level=indices


## 节点状态监控接口
- curl -XGET 127.0.0.1:9200/_nodes/stats
- curl -XGET http://127.0.0.1:9200/_cat/nodes?help

## [GC(垃圾回收)](http://kibana.logstash.es/content/elasticsearch/monitor/api/node-stats.html)
> 
对不了解 JVM 的 GC 的读者，这里先介绍一下 GC(垃圾收集)以及 GC 对 Elasticsearch 的影响。
Java is a garbage-collected language, which means that the programmer does not manually manage memory allocation and deallocation. The programmer simply writes code, and the Java Virtual Machine (JVM) manages the process of allocating memory as needed, and then later cleaning up that memory when no longer needed. Java 是一个自动垃圾收集的编程语言，启动 JVM 虚拟机的时候，会分配到固定大小的内存块，这个块叫做 heap(堆)。JVM 会把 heap 分成两个组：
Young 新实例化的对象所分配的空间。这个空间一般来说只有 100MB 到 500MB 大小。Young 空间又分为两个 survivor(幸存)空间。当 Young 空间满，就会发生一次 young gc，还存活的对象，就被移入幸存空间里，已失效的对象则被移除。
Old 老对象存储的空间。这些对象应该是长期存活而且在较长一段时间内不会变化的内容。这个空间会大很多，在 ES 来说，一节点上可能就有 30GB 内存是这个空间。前面提到的 young gc 中，如果某个对象连续多次幸存下来，就会被移进 Old 空间内。而等到 Old 空间满，就会发生一次 old gc，把失效对象移除。
听起来很美好的样子，但是这些都是有代价的！在 GC 发生的时候，JVM 需要暂停程序运行，以便自己追踪对象图收集全部失效对象。在这期间，其他一切都不会继续运行。请求没有响应，ping 没有应答，分片不会分配……
当然，young gc 一般来说执行极快，没太大影响。但是 old 空间那么大，稍慢一点的 gc 就意味着程序几秒乃至十几秒的不可用，这太危险了。
JVM 本身对 gc 算法一直在努力优化，Elasticsearch 也尽量复用内部对象，复用网络缓冲，然后还提供像 Doc Values 这样的特性。但不管怎么说，gc 性能总是我们需要密切关注的数据，因为它是集群稳定性最大的影响因子。
如果你的 ES 集群监控里发现经常有很耗时的 GC，说明集群负载很重，内存不足。严重情况下，这些 GC 导致节点无法正确响应集群之间的 ping ，可能就直接从集群里退出了。然后数据分片也随之在集群中重新迁移，引发更大的网络和磁盘 IO，正常的写入和搜索也会受到影响。

## 集群更新节点配置
* 刚上线的服务，最近一再的更新基础配置(硬盘) 

``` sh
1. 关闭分片自动均衡
PUT /_cluster/settings
{
    "transient" : {
        "cluster.routing.allocation.enable" : "none"
    }
}

2). 升级重启该节点，并确认该节点重新加入到了集群中

3). 其他节点重复第2步，升级重启。

4. 最后所有节点配置更新完成后,重启集群的shard均衡

curl -XPUT http://192.168.1.2/_cluster/settings -d'
{
    "transient" : {
        "cluster.routing.allocation.enable" : "all"
    }
}'

```



## Grafana 监控ES
- http://kibana.logstash.es/content/elasticsearch/other/grafana.html

## Kibana 2.x
* Sence
* Marvel
* 5.x 版本( x-pack ) 


转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
