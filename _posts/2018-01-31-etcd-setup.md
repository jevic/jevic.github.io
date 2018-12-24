---
layout: post
categories: Bigdata
title: etcd
date: 2018-01-31 14:39:25 +0800
description: etcd
keywords: etcd
---

etcd 是 CoreOS 团队于 2013 年 6 月发起的开源项目，它的目标是构建一个高可用的分布式键值（key-value）数据库，基于 Go 语言实现。我们知道，在分布式系统中，各种服务的配置信息的管理分享，服务的发现是一个很基本同时也是很重要的问题。CoreOS 项目就希望基于 etcd 来解决这一问题。

![](http://udn.yyuap.com/doc/docker_practice/_images/etcd_logo.png)

- 简单：支持 REST 风格的 HTTP+JSON API
- 安全：支持 HTTPS 方式的访问
- 快速：支持并发 1k/s 的写操作
- 可靠：支持分布式结构，基于 Raft 的一致性算法

## etcd

| 主机名 | ip |
| --- | --- |
| etcd1 | 192.168.3.27 |
| etcd2 | 192.168.3.28 |
| etcd3 | 192.168.3.91 |

### GitHub 下载
- [下载地址](https://github.com/coreos/etcd/releases)
- 下载解压到/usr/local/bin 即可
- 同步到所有主机
- [官网配置文档](https://coreos.com/etcd/docs/latest/op-guide/clustering.html)

### static 集群运行脚本
- 根据对应主机修改 `id` 和 `LIP`


``` sh
[root@Test-27 ~]# cat run.sh
## 根据主机名修改id
id=1  
## 修改本地监听地址
LIP=192.168.3.27
##
IP1=192.168.3.27
IP2=192.168.3.28
IP3=192.168.3.91

etcd --name etcd${id} \
-data-dir /var/lib/etcd
--initial-advertise-peer-urls http://${LIP}:2380 \
--listen-peer-urls http://0.0.0.0:2380 \
--listen-client-urls http://0.0.0.0:2379 \
--advertise-client-urls http://${LIP}:2379 \
--initial-cluster-token etcd-cluster-2 \
--initial-cluster etcd1=http://${IP1}:2380,etcd2=http://${IP2}:2380,etcd3=http://${IP3}:2380 \
--initial-cluster-state new --enable-pprof

```
- [配置详情](https://coreos.com/etcd/docs/latest/op-guide/configuration.html)


### 查看状态

``` sh
[root@Test-27 ~]# export ETCDCTL_API=3
[root@Test-27 ~]# etcdctl --write-out=table --endpoints=localhost:2379 member list
+------------------+---------+-------+--------------------------+--------------------------+
|        ID        | STATUS  | NAME  |        PEER ADDRS        |       CLIENT ADDRS       |
+------------------+---------+-------+--------------------------+--------------------------+
| 6a4616e4ba4ebc49 | started | etcd1 | http://192.168.3.27:2380 | http://192.168.3.27:2379 |
| c80e270eacac1f06 | started | etcd3 | http://192.168.3.91:2380 | http://192.168.3.91:2379 |
| d9af56dd6a49a4fc | started | etcd2 | http://192.168.3.28:2380 | http://192.168.3.28:2379 |
+------------------+---------+-------+--------------------------+--------------------------+
```

### API
具体操作参考官方文档即可

- [基本操作](https://github.com/coreos/etcd/blob/6acb3d67fbe131b3b2d5d010e00ec80182be4628/Documentation/v2/api.md)
- [集群配置](https://github.com/coreos/etcd/blob/6acb3d67fbe131b3b2d5d010e00ec80182be4628/Documentation/v2/members_api.md)
- [鉴权认证](https://github.com/coreos/etcd/blob/6acb3d67fbe131b3b2d5d010e00ec80182be4628/Documentation/v2/auth_api.md)
- [配置项](https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/configuration.md)

>示例:

```
# curl -sL http://183.129.161.81:2379/v2/keys|jq
{
  "action": "get",
  "node": {
    "dir": true,
    "nodes": [
      {
        "key": "/config",
        "dir": true,
        "modifiedIndex": 11,
        "createdIndex": 11
      },
      {
        "key": "/site",
        "dir": true,
        "modifiedIndex": 8,
        "createdIndex": 8
      }
    ]
  }
}
```

## dcmp
>提供了一个etcd的管理界面，可通过界面修改配置信息，借助confd可实现配置文件的同步

### Docker 运行
[![Docker Pulls](https://img.shields.io/docker/pulls/jevic/etcd.svg)](https://hub.docker.com/r/jevic/etcd)[![Docker Stars](https://img.shields.io/docker/stars/jevic/etcd.svg)](https://hub.docker.com/r/jevic/etcd)[![](https://images.microbadger.com/badges/image/jevic/etcd.svg)](https://microbadger.com/images/jevic/etcd "Get your own image badge on microbadger.com")

- -e ETCD_HOST etcd服务地址,默认为 127.0.0.1
- -e ROOT  配置根目录,默认为 /config 需要在etcd手动创建


``` sh
[root@master ~]# etcdctl mkdir /config
[root@master ~]# docker run -d --name dcmp -p 8000:8000 \
-e ETCD_HOST=192.168.3.27 \
jevic/etcd:dcmp
```

![](https://raw.githubusercontent.com/jevic/images/master/docker/etcd-dcmp.png)

## TLS
[TLS](https://coreos.com/os/docs/latest/generate-self-signed-certificates.html)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权


