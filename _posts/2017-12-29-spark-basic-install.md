---
layout: post
categories: Bigdata Spark
title: Spark 快速安装配置
date: 2017-01-23 16:25:46 +0800
description: Spark 快速安装配置
keywords: spark
---


## 1. 安装 spark 
下载地址 http://spark.apache.org/downloads.html
![这里写图片描述](http://img.blog.csdn.net/20161226185139368?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
> 配置Java 环境

### Master 节点：

``` sh
[spark@master ~]$ sudo tar zxf spark-2.0.2-bin-hadoop2.6.tgz -C /opt/source/
[spark@master ~]$ cd /opt/source
[spark@master ~]$ sudo mv spark-2.0.2-bin-hadoop2.6 spark
[spark@master ~]$ sudo chown spark:spark spark -R
[spark@master ~]$ cd spark/conf
[spark@master ~]$ cp spark-env.sh.template spark-env.sh

修改Spark的配置文件spark-envspark.sh
[spark@master ~]$ vim spark-env.sh
export JAVA_HOME=/opt/jdk
export SPARK_WORKER_CORES=1
export SPARK_WORKER_DIR=/home/spark/work
export SPARK_DAEMON_MEMORY=1G
export SHARK_MASTER_MEM=1G
export SPARK_MASTER_IP=192.168.11.130 

[spark@master ~]$ cp slaves.template slaves
[spark@master ~]$ cat slaves
slave 
[spark@master ~]$ scp -r /opt/source/spark slave:/home/spark
```
### Slave 节点：

``` bash
[spark@slave ~]$ sudo mv ~/spark /opt/source/  # 要注意目录权限本文统一为spark
[spark@slave source]$ ll
drwxrwxr-x 10 spark spark  150 Dec 22 13:05 spark
drwxr-xr-x 13 spark spark 4096 Dec 26 18:39 spark
```

## 2. 启动Spark集群
### 2.1 master节点：
- 启动主节点：

``` sh
[spark@master ~]$ cd /opt/source/spark/ && ./sbin/start-master.sh
```

- 启动slave节点：

``` sh
[spark@master spark]$ ./sbin/start-slave.sh 
```
![这里写图片描述](http://img.blog.csdn.net/20161226185438721?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 其他操作
* 关闭Master节点
>sbin/stop-master.sh

* 关闭Worker节点
>sbin/stop-slaves.sh

* 增加Work节点
>./sbin/start-slave.sh spark://master:7077
