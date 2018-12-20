---
layout: post
categories: ELK
title: elk Elasticsearch 管理维护操作记录 (持续更新)
date: 2017-07-21 09:26:45 +0800
description: elk
keywords: elk elasticsearch
---

# 集群信息
- 更多命令查看 [Cluster APIs](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster.html)

``` sh
curl es-90:9200/_cluster/settings?pretty
....
```


# cat 信息
- 更多命令查看 [Cat API](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat.html)

``` sh
curl es-90:9200/_cat/indices  // 索引
curl es-90:9200/_cat/shards   // 分片
curl es-90:9200/_cat/nodes?pretty  // 节点
.....

```
# node 接口信息

``` sh
curl es-90:9200/_nodes/stats/process?filter_path=**.max_file_descriptors
```

# 数据均衡
>开启: `all`
>关闭: `node`

## 开启均衡
``` sh
curl -XPUT es-90:9200/_cluster/settings?pretty -d '
{
    "transient" : {
        "cluster.routing.allocation.enable" : "all"
    }
}'

```
## 关闭均衡
``` sh
curl -XPUT es-90:9200/_cluster/settings?pretty -d '
{
    "transient" : {
        "cluster.routing.allocation.enable" : "none"
    }
}'

```

# 冷热数据分离
> 设置zone 标签，标签值自定义即可

> 此处为了方便理解配置为 热:`hot` 冷:`stale`

> 一般建议`hot`节点,磁盘使用`ssd` 提高吞吐量及效率，`stale` 则为普通`SATA`盘只用来存放历史数据

> 例如:索引名称基于日期来命名那么可将当前写入以及使用频率较高的最近时间的索引标记为`hot`，提高写入读取效率。而对应过期的冷门数据标记为`stale`

## 热节点分配

``` sh
curl -XPUT $URL/${Inx}-${date}/_settings
'{ "index.routing.allocation.include.zone": "hot"}'
```

## 冷节点分配

``` sh
curl -XPUT $URL/${Inx}-${date_ops}/_settings -d 
'{ "index.routing.allocation.include.zone": "stale"}'
```

# 索引优化分段
>将冷门数据 设置此参数提高搜索速度

## 2.x 
``` sh
curl -XPOST node-es:9200/test-2017.07.05/_optimize?max_num_segments=1&wait_for_merge=false
```

## 5.x
``` sh
curl -XPOST node-es:9200/test-2017.07.05/_forcemerge?max_num_segments=1
```
# 数据搜索

``` sh
GET index/_search
GET index/doc/_search
GET index/doc/_search?pretty
GET index/doc/_search|jq
curl node-es:9200/index/doc/_search |python -mjson.tool
```

# 搜索超时设置

``` sh
curl -XPUT es-90:9200/_cluster/settings -d '
{
    "transient" : {
  "search.default_search_timeout" : "60s"
  }
}'

```

# 数据刷新时间
> 设置为 `-1` 为不刷新
> 通常在导出数据时，建议设置为`-1`,默认设置为`60s`即可

``` sh
PUT test/_settings
 { 
    "index" : { 
        "refresh_interval" : "60s" 
    } 
}

```


# 更新文档数据
>修改某个索引指定文档的目标字段值

``` sh
POST nginx-log/logs/AVxiwZcu_Z9aU351KXOL/_update
{
  "doc": {
    "value" : 732439
  }
}

```

# 删除数据

## 删除索引

``` sh
curl -XDELETE node-es:9200/index
```
## 删除索引指定的文档

- [官方API文档](https://www.elastic.co/guide/en/elasticsearch/reference/5.5/docs-delete-by-query.html#docs-delete-by-query)


### 2.x

``` sh 
curl -XDELETE node-es:9200/test-index/_query -d '{
"query": { 
    "match": {
      "message": "some message"
    }
  }
}'
```
### 5.x
- 推荐使用: 

``` sh
curl -XPOST _delete_by_query?scroll_size=2000&wait_for_completion=false
```

``` sh
curl -XPOST node-es:9200/test-index/_delete_by_query -d '
{
  "query": { 
    "match": {
      "message": "some message"
    }
  }
}'
```


# 任务查询 _tasks
[官网API文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/tasks.html)

## es 2.x

``` sh
GET /tasks
GET _cluster/pending_tasks
GET /_tasks?actions=*/list*
GET /_tasks?actions=*reindex,*by_query
返回的结果:
{
.....
      "tasks": {
        "w8S8uYR9ThelFe7u5blX-Q:466799605": {
          "node": "w8S8uYR9ThelFe7u5blX-Q",
          "id": 466799605,
          "type": "transport",
          "action": "indices:data/write/delete/by_query",
          "start_time_in_millis": 1499846410168,
          "running_time_in_nanos": 527881720186
.....
}
```

## es 5.x 

``` sh
GET _cat/tasks?v&time
GET _tasks?detailed=true&actions=*/delete/byquery
GET _tasks?pretty&detailed&actions=*reindex,*byquery
返回结果:
{
.....
      "tasks": {
        "RiuComp3TOWhz0y804Ym7A:55597577": {
          "node": "RiuComp3TOWhz0y804Ym7A",
          "id": 55597577,
          "type": "transport",
          "action": "indices:data/write/delete/byquery",
          "status": {
            "total": 47683364,
            "updated": 0,
            "created": 0,
            "deleted": 15294000,
            "batches": 7648,
            "version_conflicts": 0,
            "noops": 0,
            .....
      }
    }
  }
}
```

## 取消任务

``` sh
POST _tasks/RiuComp3TOWhz0y804Ym7A:55597577/_cancel
```



# 模板
>模板建议使用[kopf [es 2.x]](https://github.com/lmenezes/elasticsearch-kopf) 或者[cerebro[es 5.x]](https://github.com/lmenezes/cerebro) 插件进行管理

## 添加模板

```
PUT  /_template/test
{
  
  "template":"test-*",
  "settings" : {
    "index.refresh_interval" : "60s"
  }
}
```
## 查询模板
``` sh
GET _template/test
```

# 调整数据查询窗口大小
>默认查询为1W条
>根据查询需求适当的调整此参数，不建议过大

``` sh
GET index-test/_settings

PUT index-test/_settings
{ 
  "index" : { 
    "max_result_window" : 500000 
  } 
}
```

# 磁盘配额
>此设置覆盖配置文件参数

``` sh
PUT _cluster/settings
{
    "transient" : {
        "cluster.routing.allocation.disk.watermark.low" : "90%",
        "cluster.routing.allocation.disk.watermark.high" : "10gb"
    }
}
```


# 指定索引分片分配到某个特定主机
[官网文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/shard-allocation-filtering.html)

```
PUT test/_settings
{
  "index.routing.allocation.include._ip": "192.168.2.*"
}

```

- [Index Modules](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html)

# 数据迁移导入导出
- [elasticsearch-dump](https://github.com/taskrabbit/elasticsearch-dump)

```
yum install epel-release
yum install nodejs
yum install nodejs npm
npm install elasticdump
cd node_modules/elasticdump/bin
```

- 批量执行

```
#!/bin/bash
InputES="http://es.jevic.cn"
OutputES="http://test.es.jevic.cn"

indexs="test-20181010"
for index in $indexs;do
/root/node_modules/elasticdump/bin/elasticdump \
--input=${InputES}/$index \
--output=${OutputES}/$index \
--type=data
done
```
>另外 --type=mapping 意思是把原始索引的mapping结构迁移给目标索引(2.x,5.x)

# sql 插件
- [github](https://github.com/NLPchina/elasticsearch-sql)

## sql 2.x
``` sh

curl http://es-node/_sql -d "SELECT count(*) FROM index-test/type where timestamp=1484048580"
```

## sql 5.x

``` sh
http://es-node/_sql?sql=SELECT count(*) FROM index-test where @timestamp>=149925060000 and @timestamp<1499254200000

```

# 参考链接
[Elasticsearch Reference 官网](https://www.elastic.co/guide/en/elasticsearch/reference/5.3/index.html)

[Elasticsearch: 权威指南 中文](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html)



转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
