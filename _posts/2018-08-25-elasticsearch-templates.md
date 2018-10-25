---
layout: post
categories: ELK
title: elasticsearch templates 模板配置
date: 2018-08-25 16:42:16 +0800
description: es
keywords: ELK
---

## 动态模板加载

```
{
  "order": 0,
  "template": "api-*",
  "settings": {
    "index": {
      "number_of_shards": "2",
      "number_of_replicas": "1",
      "refresh_interval": "60s"
    }
  },
  "mappings": {
    "_default_": {
      "dynamic_templates": [
        {
          "message_field": {
            "mapping": {
              "fielddata": {
                "format": "disabled"
              },
              "index": "not_analyzed",
              "omit_norms": true,
              "type": "string"
            },
            "match_mapping_type": "string",
            "match": "message"
          }
        },
        {
          "string_fields": {
            "mapping": {
              "fielddata": {
                "format": "disabled"
              },
              "index": "not_analyzed",
              "omit_norms": true,
              "type": "string",
              "fields": {
                "raw": {
                  "ignore_above": 256,
                  "index": "not_analyzed",
                  "type": "string"
                }
              }
            },
            "match_mapping_type": "string",
            "match": "*"
          }
        }
      ],
      "_all": {
        "omit_norms": true,
        "enabled": false
      },
      "properties": {
        "args": {
          "index": "not_analyzed",
          "type": "string"
        },
        "@timestamp": {
          "type": "date"
        },
        "request_time": {
          "index": "not_analyzed",
          "type": "float"
        },
        "status": {
          "index": "not_analyzed",
          "type": "long"
        }
      }
    }
  },
  "aliases": {}
}

```

## mapping 
> 适用于<6.x 版本

```
{
  "order": 0,
  "template": "test-*",
  "settings": {
    "index": {
      "number_of_shards": "3",
      "number_of_replicas": "1",
      "refresh_interval": "60s"
    }
  },
  "mappings": {
    "test1": {
      "properties": {
        "ext": {
          "index": "not_analyzed",
          "type": "string"
        },
        "id": {
          "index": "not_analyzed",
          "type": "string"
        }
      }
    },
    "test2": {
      "properties": {
        "ext": {
          "index": "not_analyzed",
          "type": "string"
        },
        "status": {
          "index": "not_analyzed",
          "type": "long"
        }
      }
    }
  },
  "aliases": {}
}

```
转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
