---
layout: post
categories: Bigdata Spark
title: Spark HA 高可用集群
date: 2017-04-01 15:54:45 +0800
description: Spark HA 高可用集群
keywords: spark 
---


## 环境准备
>Java环境
>zookeeper集群


## Master 节点操作

``` sh
[spark@master conf]$ egrep -v '^#|^$' spark-env.sh

## 注解部分
SPARK_MASTER_OPTS="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=21812 -Dcom.sun.management.jmxremote.authenticate=true  -Dcom.sun.management.jmxremote.password.file=/etc/jmx/jmx-spark.password -Dcom.sun.management.jmxremote.access.file=/etc/jmx/jmx-spark.access -Dcom.sun.management.jmxremote.ssl=false"

SPARK_WORKER_OPTS="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=21813 -Dcom.sun.management.jmxremote.authenticate=true  -Dcom.sun.management.jmxremote.password.file=/etc/jmx/jmx-spark.password -Dcom.sun.management.jmxremote.access.file=/etc/jmx/jmx-spark.access -Dcom.sun.management.jmxremote.ssl=false"

```

### 基础配置
``` bash
### zookeeper设置
export SPARK_DAEMON_JAVA_OPTS="-Dspark.deploy.recoveryMode=ZOOKEEPER -Dspark.deploy.zookeeper.url=zookeeper0.spark,zookeeper1.spark,zookeeper2.spark:2181 -Dspark.deploy.zookeeper.dir=/tspark"
export SPARK_HISTORY_OPTS="-Dspark.history.ui.port=18080 -Dspark.history.retainedApplications=3 -Dspark.history.fs.logDirectory=file:///home/spark/history"

export JAVA_HOME=/opt/jdk
export SPARK_WORKER_CORES=1
export SPARK_WORKER_DIR=/home/spark/work
export SPARK_DAEMON_MEMORY=1G
export SHARK_MASTER_MEM=1G
export SPARK_CLASSPATH=$SPARK_CLASSPATH:/opt/lib/*:/opt/spark/lib/*

[spark@master conf]$ egrep -v '^#|^$' slaves
master
slave
node1
[spark@master conf]$ scp spark-env.sh slave:/opt/spark/conf/
[spark@master conf]$ scp spark-env.sh node1:/opt/spark/conf/
[spark@master conf]$ scp slaves slave:/opt/spark/conf/
[spark@master conf]$ scp slaves node1:/opt/spark/conf/


### 启动集群
[spark@master spark]$ ./sbin/start-all.sh 
starting org.apache.spark.deploy.master.Master, logging to /opt/spark/logs/spark-spark-org.apache.spark.deploy.master.Master-1-master.out
slave: starting org.apache.spark.deploy.worker.Worker, logging to /opt/spark/logs/spark-spark-org.apache.spark.deploy.worker.Worker-1-slave.out
node1: starting org.apache.spark.deploy.worker.Worker, logging to /opt/spark/logs/spark-spark-org.apache.spark.deploy.worker.Worker-1-node1.out
master: starting org.apache.spark.deploy.worker.Worker, logging to /opt/spark/logs/spark-spark-org.apache.spark.deploy.worker.Worker-1-master.out
[spark@master spark]$ jps
20980 Master
21115 Jps
20877 Worker

```

## slave 节点启动
``` bash
[spark@slave spark]$ ./sbin/start-master.sh 
starting org.apache.spark.deploy.master.Master, logging to /opt/spark/logs/spark-spark-org.apache.spark.deploy.master.Master-1-slave.out
[spark@slave spark]$ jps
17345 Jps
17202 Worker
17271 Master

```

## 注解：
### Jconsole 监控设置
#### 认证连接：
``` bash
SPARK_MASTER_OPTS="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=21812 \
-Dcom.sun.management.jmxremote.authenticate=true \
-Dcom.sun.management.jmxremote.password.file=/etc/jmx/jmx-spark.password \
-Dcom.sun.management.jmxremote.access.file=/etc/jmx/jmx-spark.access \
-Dcom.sun.management.jmxremote.ssl=false"

```

* 密码文件示例：
/opt/jdk/jre/lib/management


#### 无认证连接：

``` sh

SPARK_MASTER_OPTS="-Dcom.sun.management.jmxremote.port=8999 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false"

```
