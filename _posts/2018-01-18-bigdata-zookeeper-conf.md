---
layout: post
categories: Bigdata
title: zookeeper
date: 2018-01-18 16:11:08 +0800
description: zookeeper
keywords: zookeeper bigdata
---

![image](http://zookeeper.apache.org/images/feather_small.gif)Apache ZooKeeper™
![image](http://zookeeper.apache.org/images/zookeeper_small.gif)
### 简介

Apache Zookeeper 是由 Apache Hadoop 的 Zookeeper 子项目发展而来，现在已经成为了 Apache 的顶级项目。Zookeeper 为分布式系统提供了高效可靠且易于使用的协同服务，它可以为分布式应用提供相当多的服务，诸如统一命名服务，配置管理，状态同步和组服务等。Zookeeper 接口简单，开发人员不必过多地纠结在分布式系统编程难于处理的同步和一致性问题上，你可以使用 Zookeeper 提供的现成(off-the-shelf)服务来实现分布式系统的配置管理，组管理，Leader 选举等功能。

Zookeeper 维护了大规模分布式系统中的常用对象，比如配置信息，层次化命名空间等，本文将从开发者的角度详细介绍 Zookeeper 的配置信息的意义以及 Zookeeper 的典型应用场景（配置文件的管理、集群管理、分布式队列、同步锁、Leader 选举、队列管理等）。


### Zookeeper 安装与配置

* 本文采用 `Zookeeper-3.4.9` 介绍它的安装步骤以及配置信息，最新的代码可以到 Zookeeper 的[官网下载](http://apache.fayea.com/zookeeper/)：     
*  Zookeeper功能强大，但是安装却十分简单，下面重点以伪分布式模式来介绍 Zookeeper 的安装。


>Zookeeper 安装模式包括：`单机模式`，`伪分布式模式`和`完全的集群模式`。
>单机模式最简单本文跳过（单机模式安装步骤参见 Zeekeeper [官方文档](http://zookeeper.apache.org/doc/current/zookeeperStarted.html)）
>伪分布式模式与集群模式配置差别不大! 为了帮助大家理解所以本文采用单台虚拟机来配置伪分布集群，
>后续大家如果熟悉后建议使用 `Docker` 提供的[官方镜像](https://hub.docker.com/)来配置集群环境

- 本文在 `CentOS 7.2` 上操作
- Java 环境为 `jdk1.8.0_111`

>安装 Zookeeper 前首先下载你需要的版本，暂时解压到指定目录
本文解压至 `/opt/source/` 目录下，本次伪分布式模拟 `3` 个 Zookeeper，目录分配如下：

``` sh
[hadoop@master source]$ mkdir /opt/source
[hadoop@master source]$ tar zxf /home/zookeeper-3.4.9.tar.gz -C /opt/source
[hadoop@master source]$ cp -a zookeeper zookeeper2 && cp -a zookeeper zookeeper3
[hadoop@master source]$ tree -d -L 1
.
├── zookeeper
├── zookeeper2
└── zookeeper3
[hadoop@master ~]$ tree -d -L 1 /data
/data
├── zookeeper
├── zookeeper2
└── zookeeper3
```

#### Zookeeper 配置文件
* 这里我们只介绍 zoo_sample.cfg，首先将此文件重命名为`zoo.cfg`，并修改对应目录下的zoo.cfg。整体配置其实很简单所以这里只列出了配置后启用参数，具体参数的说明请参考官方文档这里不在叙述：

``` sh
[hadoop@master source]$ egrep -v "^#|^$" zookeeper/conf/zoo.cfg 
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/data/zookeeper
clientPort=2181
server.1=master:3331:4881
server.2=master:3332:4882
server.3=master:3333:4883
[hadoop@master source]$ egrep -v "^#|^$" zookeeper2/conf/zoo.cfg 
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/data/zookeeper2
clientPort=2182
server.1=master:3331:4881
server.2=master:3332:4882
server.3=master:3333:4883
[hadoop@master source]$ egrep -v "^#|^$" zookeeper3/conf/zoo.cfg 
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/data/zookeeper3
clientPort=2183
server.1=master:3331:4881
server.2=master:3332:4882
server.3=master:3333:4883

[hadoop@master ~]$ echo "1" > /data/zookeeper/myid
[hadoop@master ~]$ echo "2" > /data/zookeeper2/myid
[hadoop@master ~]$ echo "3" > /data/zookeeper3/myid

```

#### 启动服务

``` sh
[hadoop@master ~]$ /opt/source/zookeeper/bin/zkServer.sh start
[hadoop@master ~]$ /opt/source/zookeeper2/bin/zkServer.sh start
[hadoop@master ~]$ /opt/source/zookeeper3/bin/zkServer.sh start
```

#### 参数说明
* tickTime：这个时间是作为 Zookeeper 服务器之间或客户端与服务器之间维持心跳的时间间隔，也就是每个 tickTime 时间就会发送一个心跳。
* dataDir：顾名思义就是 Zookeeper 保存数据的目录，默认情况下，Zookeeper 将写数据的日志文件也保存在这个目录里。
* clientPort：这个端口就是客户端连接 Zookeeper 服务器的端口，Zookeeper 会监听这个端口，接受客户端的访问请求。
* initLimit：这个配置项是用来配置 Zookeeper 接受客户端
    * （这里所说的客户端不是用户连接 Zookeeper 服务器的客户端，
  而是 Zookeeper 服务器集群中连接到 Leader 的 Follower 服务器）初始化连接时最长能忍受多少个心跳时间间隔数。
    * 当已经超过 5个心跳的时间（也就是 tickTime）长度后 Zookeeper 服务器还没有收到客户端的返回信息，那么表明这个客户端连接失败。总的时间长度就是 5*2000=10 秒
* syncLimit：这个配置项标识 Leader 与Follower 之间发送消息，请求和应答时间长度，最长不能超过多少个 tickTime 的时间长度，总的时间长度就是2*2000=4 秒
server.A=B：C：D：
    * 其中 A 是一个数字，表示这个是第几号服务器；
    * B 是这个服务器的 ip 地址；
    * C 表示的是这个服务器与集群中的 Leader 服务器交换信息的端口；
    * D 表示的是万一集群中的 Leader 服务器挂了，需要一个端口来重新进行选举，选出一个新的 Leader，而这个端口就是用来执行选举时服务器相互通信的端口。
    * 如果是伪集群的配置方式，由于 B 都是一样，所以不同的 Zookeeper 实例通信端口号不能一样，所以要给它们分配不同的端口号

### 服务查看
``` sh
[hadoop@master ~]$ netstat -tnlp |grep 'java'
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 192.168.11.130:4881     0.0.0.0:*               LISTEN      3751/java           
tcp        0      0 192.168.11.130:4882     0.0.0.0:*               LISTEN      3779/java           
tcp        0      0 192.168.11.130:4883     0.0.0.0:*               LISTEN      3820/java           
tcp        0      0 0.0.0.0:52021           0.0.0.0:*               LISTEN      3820/java           
tcp        0      0 0.0.0.0:42968           0.0.0.0:*               LISTEN      3751/java           
tcp        0      0 0.0.0.0:45241           0.0.0.0:*               LISTEN      3779/java           
tcp        0      0 192.168.11.130:3332     0.0.0.0:*               LISTEN      3779/java           
tcp        0      0 0.0.0.0:2181            0.0.0.0:*               LISTEN      3751/java           
tcp        0      0 0.0.0.0:2182            0.0.0.0:*               LISTEN      3779/java           
tcp        0      0 0.0.0.0:2183            0.0.0.0:*               LISTEN      3820/java

```

### 验证测试

``` sh
[hadoop@master ~]$ /opt/source/zookeeper/bin/zkCli.sh -server master:2181
Welcome to ZooKeeper!
JLine support is enabled
2016-12-26 15:20:48,698 [myid:] - INFO  [main-SendThread(master:2181):ClientCnxn$SendThread@876] - Socket conn
ection established to master/192.168.11.130:2181, initiating session2016-12-26 15:20:48,725 [myid:] - INFO  [main-SendThread(master:2181):ClientCnxn$SendThread@1299] - Session es
tablishment complete on server master/192.168.11.130:2181, sessionid = 0x15939ddfedb0001, negotiated timeout = 30000
WATCHER::
WatchedEvent state:SyncConnected type:None path:null
[zk: master:2181(CONNECTED) 1] create /test 123

另开一个终端：
[root@master ~]# /opt/source/zookeeper3/bin/zkCli.sh -server master:2183
Welcome to ZooKeeper!
JLine support is enabled
2016-12-26 15:21:40,669 [myid:] - INFO  [main-SendThread(master:2183):ClientCnxn$SendThread@876] - Socket conn
ection established to master/192.168.11.130:2183, initiating session2016-12-26 15:21:40,691 [myid:] - INFO  [main-SendThread(master:2183):ClientCnxn$SendThread@1299] - Session es
tablishment complete on server master/192.168.11.130:2183, sessionid = 0x35939de0e410001, negotiated timeout = 30000
WATCHER::
WatchedEvent state:SyncConnected type:None path:null
[zk: master:2183(CONNECTED) 1] get /test
123
cZxid = 0x100000009
ctime = Mon Dec 26 15:21:12 CST 2016
mZxid = 0x100000009
mtime = Mon Dec 26 15:21:12 CST 2016
pZxid = 0x100000009
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 4
numChildren = 0
```

更多详细信息请查看[官方文档](http://zookeeper.apache.org/doc/r3.4.9/)


转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
