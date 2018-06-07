---
layout: post
categories: ELK
title: elk Elasticsearch 分片丢失问题处理
date: 2018-01-23 21:58:39 +0800
description: elk Elasticsearch 分片丢失问题处理
keywords: elk es
---

# 问题概述
>由于集群节点下线更换操作，导致了部分索引主分片丢失，集群处于red状态；
>而且此时集群分片均衡失效，不在均衡数据；
>尝试使用[reroute API](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-reroute.html) 接口来手动迁移，发现命令执行失败；
>接下来就比较尴尬了....均衡不了....手动迁移也失败.... 
>综合以上状况只能做以下操作了...

# 处理过程
>将处于`red`状态的并且过期不在使用的索引删除(此乃下下策😭)
>保留并处理热门数据

## 1. 重建索引

``` sh
curl -XPOST es-90:9200/_reindex -d '
{
  "source": {
    "index": "index-name"
  },
  "dest": {
    "index": "index-name.bak"
  }
}'

```

## 2. 删除旧索引

``` sh
curl -XDELETE es-90:9200/index-name

```

## 3. 建立别名

``` sh
curl -XPUT es-90:9200/index-name.bak/_alias/index-name
```


# 批量处理

``` sh

#!/bin/bash
indexs="index1 index2 index3"

back(){
curl -XPOST es-90:9200/_reindex -d '
{
  "source": {
    "index": "'$index'"
  },
  "dest": {
    "index": "'${index}.bak'"
  }
}'
}

for index in $indexs;do
    back
    curl -XDELETE es-90:9200/${index}
    curl -XPUT es-90:9200/${index}.bak/_alias/${index}
done

```

# 副本丢失
> 当索引副本分片丢失 且无法正常均衡时

## 1. 取消副本

``` sh
PUT /test/_settings
{
    "index" : {
        "number_of_replicas" : 0
    }
}

```

## 2. 恢复副本

``` sh

PUT /test/_settings
{
    "index" : {
        "number_of_replicas" : 1
    }
}

```



转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
