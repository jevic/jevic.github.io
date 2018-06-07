---
layout: post
categories: Kafka
title: Kafka Zookeeper 操作杂录
date: 2017-09-12 17:35:11 +0800
description: Kafka Zookeeper 操作杂录
keywords: kafka zookeeper
---

>zookeeper kafka 杂录
>日常使用操作记录及故障处理记录


## zookeeper
### 可视化日志

```
java -cp zookeeper-3.4.9.jar:lib/slf4j-api-1.6.1.jar:lib/slf4j-log4j12-1.6.1.jar:lib/log4j-1.2.16.jar:conf org.apache.zookeeper.server.LogFormatter /data/zookeeper/version-2/log.100000001
```

### 可视化快照

```
java -cp zookeeper-3.4.9.jar:lib/slf4j-api-1.6.1.jar org.apache.zookeeper.server.SnapshotFormatter /data/zookeeper/version-2/snapshot.100000030

```

### 数据清理

``` sh
1. 使用自带命令
zkCleanup.sh /datac/zk_data/version-2  20
/datac/zk_data/version-2 指的是存放快照和log文件的目录
20 代表保留最近20份数据

2. 设定定时任务脚本
保留最新的66个文件
#!/bin/bash 
#snapshot file dir 
dataDir=/home/test/zk_data/version-2 
#tran log dir 
dataLogDir=/home/test/zk_log/version-2 
#zk log dir 
logDir=/home/test/logs 
#Leave 66 files 
count=66 
count=$[$count+1] 
ls -t $dataLogDir/log.* | tail -n +$count | xargs rm -f 
ls -t $dataDir/snapshot.* | tail -n +$count | xargs rm -f 
ls -t $logDir/zookeeper.log.* | tail -n +$count | xargs rm -f


3. 配置文件中指定参数
autopurge.purgeInterval  这个参数指定了清理频率，单位是小时，需要填写一个1或更大的整数，默认是0，表示不开启自己清理功能。
autopurge.snapRetainCount 这个参数和上面的参数搭配使用，这个参数指定了需要保留的文件数目。默认是保留3个
```


----


## kafka

``` sh
> bin/kafka-console-consumer.sh --zookeeper localhost:2181/kafka --from-beginning --topic test

2.10
> ./kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test
> ./kafka-console-producer.sh --broker-list localhost:9092 --topic test


> bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
> bin/kafka-topics.sh --list --zookeeper localhost:2181
> bin/kafka-topics.sh --delete --topic ngx-logs --zookeeper kafka1:2181
> bin/kafka-consumer-groups.sh --bootstrap-server broker1:9092 --list logstash

```


## kafka 集群扩展:
[官方文档](http://kafka.apache.org/documentation/#basic_ops_automigrate)

```

--generate: 生成
--execute:  执行
--verify:   查看

1. 编辑topic 分区文件
  > cat topics-to-move.json
  {"topics": [{"topic": "foo1"},
              {"topic": "foo2"}],
  "version":1
  }

2. 生成分布文件
[root@kafka1 kafka]# ./bin/kafka-reassign-partitions.sh --zookeeper kafka1:2181 --topics-to-move-json-file topics-to-move.json --broker-list "3,4" --generate
Current partition replica assignment

{"version":1,"partitions":[{"topic":"nginx302a","partition":4,"replicas":[1,3]},{"topic":"nginx302a","partition":3,"replicas":[0,2]},{"topic":"nginx302a","partition":0,"replicas":[2,4]},{"topic":"nginx302a","partition":1,"replicas":[3,0]},{"topic":"nginx302a","partition":2,"replicas":[4,1]}]}
Proposed partition reassignment configuration  [将下面部分复制保存到文件]

{"version":1,"partitions":[{"topic":"nginx302a","partition":4,"replicas":[3,4]},{"topic":"nginx302a","partition":3,"replicas":[4,3]},{"topic":"nginx302a","partition":0,"replicas":[3,4]},{"topic":"nginx302a","partition":1,"replicas":[4,3]},{"topic":"nginx302a","partition":2,"replicas":[3,4]}]}


3. 执行移动分片
 > bin/kafka-reassign-partitions.sh --zookeeper localhost:2181 --reassignment-json-file expand-cluster-reassignment.json --execute

4.  查看运行结果 
> bin/kafka-reassign-partitions.sh --zookeeper localhost:2181 --reassignment-json-file expand-cluster-reassignment.json --verify


对于分区文件可以手动定义

```

## 文件系统

- Ext4
1. data=writeback
2. Disabling journaling
3. commit=num_secs
4. nobh
5. delalloc



## 示例:

```
./bin/kafka-reassign-partitions.sh --zookeeper kafka0.yfcloud:2181,kafka1.yfcloud:2181,kafka2.yfcloud:2181,kafka3.yfcloud:2181,kafka4.yfcloud:2181/kafka --reassignment-json-file reass.json --verify


./bin/kafka-topics.sh --describe --topic ngx302 --zookeeper kafka0.yfcloud:2181,kafka1.yfcloud:2181,kafka2.yfcloud:2181,kafka3.yfcloud:2181,kafka4.yfcloud:2181/kafka

``` 


## 集群数据均衡失效处理
```
kafka manager:
Yikes! Partition reassignment currently in progress for

get /kafka/admin/reassign_partitions
rmr /kafka/admin/reassign_partitions
```

