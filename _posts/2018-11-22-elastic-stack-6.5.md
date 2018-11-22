---
layout: post
categories: ELK
title: elastic-stack 6.5
date: 2018-11-22 19:56:06 +0800
description: elk,elasticsearch
keywords: es,kibana
---

> - 在以往的旧版本(2.x,5.x) 每个索引可以存储不同类型的文档,
    -  类比MySQL
        - index == database
        - _type == table
- 6.x 版本开始 移除了_type  也就是每个索引只有一种类型！！！
- x-pack 从6.3版本开始已经内置在elasticsearch,kibana 当中无需另行安装!

### elasticsearch

- [官网下载](https://www.elastic.co/downloads)

- 下载解压

- 配置基础环境

  ```
  # cat /etc/security/limit.conf
  * soft nofile 65536
  * hard nofile 65536
  * soft nproc unlimited
  * hard nproc unlimited
  es soft memlock unlimited
  es hard memlock unlimited
  
  # cat /etc/sysctl.conf
  vm.swappiness = 1
  vm.max_map_count=262144
  ```

- 添加 es 用户并授权

- 启动

- 授权 license （30天试用版）

    - 证书申请: https://register.elastic.co/marvel_register   

```
curl -H "Content-Type:application/json" -XPOST  http://192.168.2.221:9200/_xpack/license/start_trial?acknowledge=true
```
### x-pack 开启认证
- 配置用户名密码:

  - $ES_PATH:/bin/elasticsearch-setup-passwords interactive 
  - 根据提示一步步设置密码即可

- 修改 elasticsearch 配置文件

  ```
  # tail elasticsearch.yml -n 1
  xpack.security.enabled: true  ## 开启认证
  ```

- 重启ES，再次访问则需要输入用户名密码

- 修改密码

```
curl -H "Content-Type:application/json" -XPOST -u elastic 'http://192.168.2.221:9200/_xpack/security/user/elastic/_password' -d '{ "password" : "123456" }'
```

### kibana

- 配置对应的用户名密码以及 ES 链接地址
- 配置文件添加此配置: 
  - xpack.security.enabled: true
- 启动即可

## cerebro 

- [github](https://github.com/lmenezes/cerebro)
- 具体步骤查看文档即可

## sql 插件

- [github](https://github.com/NLPchina/elasticsearch-sql)


转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
