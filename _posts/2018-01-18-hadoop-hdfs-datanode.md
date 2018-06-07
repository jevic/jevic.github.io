---
layout: post
categories: Bigdata
title: hadoop HDFS Datanode 节点动态扩展(二)
date: 2018-01-18 15:37:37 +0800
description: hadoop HDFS Datanode 节点动态扩展(二)
keywords: hdfs hadoop
---

## 扩容 Datanode
1. 增加hostname,同步/etc/hosts 到所有节点,配置双机互信
2. 添加主机名到主节点的 slave 文件中
3. 同步主节点Hadoop 运行目录到新增Datanode 节点 并修改权限
4. 创建数据目录并修改权限
5. 启动datanode


### 配置互信 hostname slave

``` sh
[hadoop@hdfs-node3 .ssh]$ pwd
/home/hadoop/.ssh
[hadoop@hdfs-node3 .ssh]$ ls
authorized_keys  id_rsa 

[hadoop@hdfs-master ~]# cat /etc/hosts |grep hdfs
192.168.3.27 hdfs-master
192.168.3.28 hdfs-slave
192.168.3.91 hdfs-node3
[hadoop@hdfs-master ~]# scp /etc/hosts hdfs-slave:/etc/hosts
[hadoop@hdfs-master ~]# scp /etc/hosts hdfs-node3:/etc/hosts
[hadoop@hdfs-master ~]# cat /usr/local/hadoop/etc/hadoop/slaves
hdfs-master
hdfs-slave
hdfs-node3
[hadoop@hdfs-master ~]# scp /usr/local/hadoop/etc/hadoop/slaves hdfs-slave:/usr/local/hadoop/etc/hadoop/slaves

```
### 同步Hadoop运行目录
```
[hadoop@hdfs-master ~]# scp /usr/local/hadoop hdfs-node3:/usr/local/

```

### 创建目录并修改权限

``` sh 
[root@hdfs-node3 local]# chown -R hadoop.hadoop hadoop
[root@hdfs-node3 local]# mkdir -p /data/hadoop/data/{dfs,tmp,yarn}
[root@hdfs-node3 local]# mkdir -p /data/hadoop/data/dfs/data
[root@hdfs-node3 local]# chown -R hadoop.hadoop /data/hadoop

```

### 启动 datanode

``` sh
[hadoop@hdfs-node3 hadoop]$ ./sbin/hadoop-daemon.sh start datanode
```

### master节点查看当前集群信息 确认新节点已经加入

``` sh
[hadoop@hdfs-master ~]$ hdfs dfsadmin -report
Configured Capacity: 548221317120 (510.57 GB)
Present Capacity: 446565990400 (415.90 GB)
DFS Remaining: 446516936704 (415.85 GB)
DFS Used: 49053696 (46.78 MB)
DFS Used%: 0.01%
Under replicated blocks: 0
Blocks with corrupt replicas: 0
Missing blocks: 0
Missing blocks (with replication factor 1): 0

-------------------------------------------------
Live datanodes (3):

Name: 192.168.3.27:50010 (hdfs-master)
Hostname: hdfs-master
Decommission Status : Normal
Configured Capacity: 440899563520 (410.62 GB)
DFS Used: 16351232 (15.59 MB)
Non DFS Used: 21774798848 (20.28 GB)
DFS Remaining: 419108413440 (390.33 GB)
DFS Used%: 0.00%
DFS Remaining%: 95.06%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Tue Jan 16 14:03:18 CST 2018


Name: 192.168.3.91:50010 (hdfs-node3)
Hostname: hdfs-node3
Decommission Status : Normal
Configured Capacity: 53660876800 (49.98 GB)
DFS Used: 16351232 (15.59 MB)
Non DFS Used: 37875245056 (35.27 GB)
DFS Remaining: 15769280512 (14.69 GB)
DFS Used%: 0.03%
DFS Remaining%: 29.39%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Tue Jan 16 14:03:16 CST 2018


Name: 192.168.3.28:50010 (hdfs-slave)
Hostname: hdfs-slave
Decommission Status : Normal
Configured Capacity: 53660876800 (49.98 GB)
DFS Used: 16351232 (15.59 MB)
Non DFS Used: 42005282816 (39.12 GB)
DFS Remaining: 11639242752 (10.84 GB)
DFS Used%: 0.03%
DFS Remaining%: 21.69%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Tue Jan 16 14:03:18 CST 2018

```

### master节点开启负载均衡

``` sh
[hadoop@hdfs-master hadoop]$ hdfs dfsadmin -setBalancerBandwidth 67108864
[hadoop@hdfs-master hadoop]$ start-balancer.sh -threshold 5
starting balancer, logging to /usr/local/hadoop/logs/hadoop-hadoop-balancer-hdfs-master.out
Time Stamp               Iteration#  Bytes Already Moved  Bytes Left To Move  Bytes Being Moved
```


### 新节点启动nodemanager

```sh
[hadoop@hdfs-node3 hadoop]$ ./sbin/yarn-daemon.sh start nodemanager
```

## Datanode 节点退役

> hdfs-site.xml 必须配置此部分 

``` sh
[hadoop@hdfs-master hadoop]$ tail -n 5 hdfs-site.xml
　　<property>
　　    <!--dfs.hosts.exclude定义的文件内容为,每个需要下线的机器，一行一个-->
        <name>dfs.hosts.exclude</name>
　      <value>/usr/local/hadoop/etc/hadoop/excludes</value>
    </property>
</configuration>

```

### excludes 文件
>在 $HADOOP_HOME/etc/hadoop/excludes 中配置想要退役的节点，一行一个
>如果配置了HA 则需要同步到standby 节点

```
[hadoop@hdfs-master hadoop]$ scp excludes hdfs-slave:/usr/local/hadoop/etc/hadoop/
[hadoop@hdfs-master hadoop]$ cat excludes
hdfs-node4
```

> 然后在namenode 节点执行 刷新

``` sh
[hadoop@hdfs-master ~]$ hdfs dfsadmin -refreshNodes
```

- 查看web 界面发现节点已经退役状态

### 恢复节点

>当退役节点维护完毕 需要重新恢复到集群时

- 重启datanode
- 查看并确认 DataNode节点已经加入
- 启动 nodemanager 即可


>注意: 当启动或重启 `namenode` 节点时务必把 `excludes` 文件清空后再进行操作
>一定要注意操作的顺序 否则会报错


