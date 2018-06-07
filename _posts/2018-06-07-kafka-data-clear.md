---
layout: post
categories: Kafka
title: kafka 数据保存时间动态调整
date: 2018-06-07 22:07:44 +0800
description: Kafka
keywords: Kafka
---

> 动态调整kafka 数据保存时间,清理过期数据!!


```
#!/bin/bash
## kafka 版本: kafka_2.10-0.8.2.2
topics="test1 test2"
zk="zk1:2181,zk2:2181,zk3:2181/kafka"
### 修改保留时间
### 保留几个小时
hours=2
### 转换为毫秒
Times=`echo "$hours * 3600000"|bc`

## 修改配置
### 控制未压缩数据 retention.ms
conf="retention.ms"
### 控制压缩后的数据 delete.retention.ms
#conf="delete.retention.ms"

ClearLog(){
## 清理数据
for i in $topics;do
echo "/opt/kafka/bin/kafka-topics.sh --zookeeper $zk --alter --topic $i --config ${conf}=${Times}"
#/opt/kafka/bin/kafka-topics.sh --zookeeper $zk --alter --topic $i --config ${conf}=${Times}
done
}

DelConf(){
## 删除配置
echo "/opt/kafka/bin/kafka-topics.sh --zookeeper $zk --alter --topic $i --delete-config $conf"
#/opt/kafka/bin/kafka-topics.sh --zookeeper $zk --alter --topic $i --delete-config $conf
}

case $1 in
     clear)
     ClearLog ;;
     delconf)
     DelConf ;;
     *)
     echo "clear --- 清理日志"
     echo "delconf --- 删除配置"
     ;;
esac

```


转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
