---
layout: post
categories: ELK
title: Elasticsearch 索引分片维护操作
date: 2017-05-16 17:57:23 +0800
description: Elasticsearch 索引分片维护操作
keywords: elasticsearch elk ELK
---


>索引分片移动 集群数据维护操作


### 索引创建

``` sh
# curl -XPOST "http://ESnode:9200/man'
# curl -XPUT "http://ESnode:9200/man/_settings' -d '{
   "index.routing.allocation.include.zone" : "zone_one"
   }'
```
   
>第一条命令是创建man索引；
第二条命令是发送到_settings REST端点，用来指定这个索引的其他配置信息。我们将index.routing.allocation.include.zone属性设置为zone_one值，就是我们所希望的把man索引放置到node.zone属性值为zone_one的ES集群节点服务器上。

同样对woman索引我们做类似操作：

``` sh
# curl -XPOST "http://ESnode:9200/woman'
# curl -XPUT "http://ESnode:9200/woman/_settings' -d '{
   "index.routing.allocation.include.zone" : "zone_two"
   }'
```
不同的是，这次指定woman索引放置在node.zone属性值为zone_two的ES集群节点服务器上
 
 
最后我们需要将katoey索引放置到上面所有的ES集群节点上面，配置设置命令如下：

``` sh
# curl -XPOST "http://ESnode:9200/katoey"
# curl -XPUT "http://ESnode:9200/katoey/_settings" -d '{
  "index.routing.allocation.include.zone" : "zone_one,zone_two"
  }'

```
  
### 分配时排除节点
>跟我们上面操作为索引指定放置节点位置一样，我们也可以在索引分配的时候排除某些节点。
参照之前的例子，我们新建一个people索引，但是不希望people索引放置到zone_one的ES集群节点服务器上，我们可以运行如下命令操作：

```
# curl -XPOST "http://EScode:9200/people"
# curl -XPUT "http://EScode:9200/people/_settings" -d '{
  "index.routing.allocation.exclude.zone" : "zone_one"
  }'
请注意，在这里我们使用的是index.routing.allocation.exclude.zone属性
而不是index.routing.allocation.include.zone属性。
```


### 使用IP地址进行分配配置

>除了在节点的配置中添加一些特殊的属性参数外，我们还可以使用IP地址来指定你将分片和副本分配或者不分配到哪些节点上面。
>为了做到这点，我们应该使用_ip属性，把zone换成_ip就好了。
>例如我们希望lucky索引分配到IP地址为10.0.1.110和10.0.1.119的节点上，我们可以运行如下命令设置：

```
# curl -XPOST "http://ESnode:9200/lucky"
# curl -XPUT "http://ESnode:9200/lucky/_settings" -d '{
  "index.routing.allocation.include._ip" "10.0.1.110,10.0.1.119"
  }'
```



### 集群范围内分配

- 除了索引层面指定分配活着排除分配之外(上面我们所做的都是这两种情况),我们还可以指定集群中所有索引的分配。
- 例如，我们希望将所有的新索引分配到IP地址为10.0.1.112和10.0.1.114的节点上，我们可以运行如下命令设置：
    
``` sh
# curl -XPUT "http://ESnode:9200/_cluster/settings" -d '{
  "transient" : {
   "cluster.routing.allocation.include._ip" "10.0.1.112,10.0.1.114"
   }
  }'
  
```
>集群级别的控制还有transient和persistent属性


### 每个节点上分片和副本数量的控制

- 除了指定分片和副本的分配，我们还可以对一个索引指定每个节点上的最大分片数量。
- 例如我们希望ops索引在每个节点上只有一个分片，我们可以运行如下命令：
    
``` sh
  # curl -XPUT "http://ESnode:9200/ops/_settings" -d '{
  "index.routing.allocation.total_shards_per_node" : 1
  }'
  
```
>这个属性也可以直接配置到elasticsearch.ym配置文件中，或者使用上面命令在活动索引上更新。
>如果配置不当，导致主分片无法分配的话，集群就会处于red状态。

### 手动移动分片和副本
>接下来我们介绍一下节点间手动移动分片和副本。可以使用ElasticSearch提供的_cluster/reroute REST端点进行控制，能够进行下面操作：

- 取消对分片的分配
- 强制对分片进行分配
- 移动分片

#### 将一个分片从一个节点移动到另外一个节点
>假设我们有两个节点：es_node_one和es_node_two，ElasticSearch在es_node_one节点上分配了ops索引的两个分片，我们现在希望将第二个分片移动到es_node_two节点上。可以如下操作实现：

``` sh
# curl -XPOST "http://ESnode:9200/_cluster/reroute' -d  '{
   "commands" : [ {
   "move" : {
   "index" : "ops",
   "shard" : 1,
   "from_node" : "es_node_one",
   "to_node" : "es_node_two"
   }
  }]
  }'
  
我们通过move命令的index属性指定移动哪个索引,通过shard属性指定移动哪个分片，
最终通过from_node属性指定我们从哪个节点上移动分片，
通过to_node属性指定我们希望将分片移动到哪个节点。

```
    

 
#### 取消分配
>如果希望取消一个正在进行的分配过程，我们通过运行cancel命令来指定我们希望取消分配的索引、节点以及分片，如下所示：
    
``` sh
# curl -XPOST "http://ESnode:9200/_cluster/reroute" -d '{
  "commands" : [ {
  "cancel" : {
  "index" : "ops",
  "shard" : 0,
  "node" : "es_node_one"
  }
  } ]
  }'
运行上面的命令将会取消es_node_one节上ops索引的第0个分片的分配

```

 
#### 分配分片
>除了取消和移动分片和副本之外，我们还可以将一个未分配的分片分配到一个指定的节点上。
假设ops索引上有一个编号为0的分片尚未分配，并且我们希望ElasticSearch将其分配到es_node_two上，可以运行如下命令操作：

    
``` sh

# curl -XPOST "http://ESnode:9200/_cluster/reroute' -d '{
  "commands" : [ {
   "allocate" : {
    "index" : "ops",
    "shard" : 0,
    "node" : "es_node_two"
    }
   } ]
   }'

一次HTTP请求包含多个命令
我们可以在一次HTTP请求中包含多个命令，例如：

# curl -XPOST "http://ESnode:9200/_cluster/reroute" -d '{
   "commands" : [
     {"move" : 
            {"index" : "ops", 
        "shard" : 1, 
        "from_node" : "es_node_one", 
        "to_node" : "es_node_two"
        }
     },
     {"cancel" : 
        {"index" : "ops", 
     "shard" : 0, 
     "node" : "es_node_one"}
     }
    ]
 }'

```



