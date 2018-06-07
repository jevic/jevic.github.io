---
layout: post
categories: Bigdata
title: hadoop HDFS HA 高可用配置(一)
date: 2018-01-18 15:35:27 +0800
description: hadoop HDFS HA 高可用配置(一)
keywords: hadoop hdfs bigdata
---

![Hadoop](http://hadoop.apache.org/images/hadoop-logo.jpg)

## 概述
* Hadoop是一个由Apache基金会所开发的分布式系统基础架构。
用户可以在不了解分布式底层细节的情况下，开发分布式程序。充分利用集群的威力进行高速运算和存储。
* Hadoop实现了一个分布式文件系统（Hadoop Distributed File System），简称HDFS。HDFS有高容错性的特点，并且设计用来部署在低廉的（low-cost）硬件上；而且它提供高吞吐量（high throughput）来访问应用程序的数据，适合那些有着超大数据集（large data set）的应用程序。
* HDFS放宽了（relax）POSIX的要求，可以以流的形式访问（streaming access）文件系统中的数据。
* Hadoop的框架最核心的设计就是：HDFS和MapReduce。
* HDFS为海量的数据提供了存储，则MapReduce为海量的数据提供了计算。

>Hadoop2.x之后的版本提供了解决单点问题的方案－HA(HighAvailable 高可用) 执行步骤如下：
 

### 配置 HDFS HA
#### 依赖组件下载
- [JDK](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
- [zookeeper](http://zookeeper.apache.org/releases.html)
- [hadoop2.x](http://apache.fayea.com/hadoop/common/)

#### 初始化主机
1. hostname

``` sh
192.168.3.27 hdfs-master
192.168.3.28 hdfs-slave

```

2. 关闭防火墙 selinux

3. 创建 hadoop 用户

4. ulimit

``` sh
# cat /etc/security/limits.conf
* soft nofile 102400
* hard nofile 102400
```

5. 配置 hadoop 用户双机互信

``` sh
[hadoop@hdfs-master ~]$ 
[hadoop@hdfs-master ~]$ ssh-keygen -t rsa
[hadoop@hdfs-master ~]$ cat .ssh/id_rsa.pub >> .ssh/authorized_keys
[hadoop@hdfs-master ~]$ chmod 600 .ssh/authorized_keys
[hadoop@hdfs-master ~]$ 
复制master 公钥到各节点 步骤略.......
[hadoop@hdfs-master hadoop]$
```

#### jdk
- /opt/jdk

``` sh
cat /etc/profile
export JAVA_HOME=/opt/jdk
export PATH=${JAVA_HOME}/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/jre/lib/dt.jar:$JAVA_HOME/lib/tools.jar

```

#### zookeeper
- [zookeeper 配置实例](http://www.jevic.cn/categories/#Bigdata)



#### Hadoop
- /usr/local/hadoop

##### 创建目录(所有节点)

```sh
# mkdir -p /data/hadoop/data/{dfs,tmp,yarn}
# mkdir -p /data/hadoop/data/dfs/{data,name}
# chown -R hadoop.hadoop /data/hadoop
```

##### 编辑配置文件

###### 切换到hadoop 用户操作

``` sh
[root@hdfs-master ~]# su - hadoop
[hadoop@hdfs-master ~]$ cd /usr/local/hadoop/etc/hadoop/
```
###### hadoop-env.sh

``` sh
export JAVA_HOME=/opt/jdk
#export HADOOP_SSH_OPTS="-p 22" 默认为22

```
###### slave

``` sh
[hadoop@hdfs-master hadoop]$ cat slaves
hdfs-master
hdfs-slave
```

###### core-site.yml

``` sh
[hadoop@hdfs-master hadoop]$ cat core-site.xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
<property>
        <name>fs.defaultFS</name>
        <value>hdfs://hadoop-node</value>
    </property>
    <property>
        <name>io.file.buffer.size</name>
        <value>131072</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/data/hadoop/tmp</value>
    </property>
    <property>
        <name>hadoop.proxyuser.hduser.hosts</name>
        <value>*</value>
    </property>
    <property>
        <name>hadoop.proxyuser.hduser.groups</name>
        <value>*</value>
    </property>
    <property>
        <name>ha.zookeeper.quorum</name>
        <value>hdfs-master:2181</value>
    </property>
</configuration>

```

###### hdfs-site.xml

``` sh
[hadoop@hdfs-master hadoop]$ cat hdfs-site.xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>dfs.nameservices</name>
        <value>hadoop-node</value>
    </property>
    <property>
        <name>dfs.ha.namenodes.hadoop-node</name>
        <value>nmaster,nslave</value>
    </property>
    <property>
        <name>dfs.namenode.rpc-address.hadoop-node.nmaster</name>
        <value>hdfs-master:9000</value>
    </property>
    <property>
        <name>dfs.namenode.rpc-address.hadoop-node.nslave</name>
        <value>hdfs-slave:9000</value>
    </property>

    <property>
        <name>dfs.namenode.http-address.hadoop-node.nmaster</name>
        <value>hdfs-master:50070</value>
    </property>

    <property>
        <name>dfs.namenode.http-address.hadoop-node.nslave</name>
        <value>hdfs-slave:50070</value>
    </property>
    <property>
        <name>dfs.namenode.shared.edits.dir</name>
        <value>qjournal://hdfs-master:8485;hdfs-slave:8485/hadoop-node</value>
    </property>

    <property>
        <name>dfs.client.failover.proxy.provider.hadoop-node</name>
        <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
    </property>
    <property>
        <name>dfs.ha.fencing.methods</name>
        <value>sshfence</value>
    </property>
    <property>
        <name>dfs.ha.fencing.ssh.private-key-files</name>
        <value>/home/hadoop/.ssh/id_rsa</value>
    </property>
    <property>
        <name>dfs.journalnode.edits.dir</name>
        <value>/data/hadoop/data/tmp/journal</value>
    </property>
    <property>
        <name>dfs.ha.automatic-failover.enabled</name>
        <value>true</value>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>/data/hadoop/data/dfs/name</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>/data/hadoop/data/dfs/data</value>
    </property>
    <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>
    <property>
        <name>dfs.webhdfs.enabled</name>
        <value>true</value>
    </property>
    <property>
        <name>dfs.journalnode.http-address</name>
        <value>0.0.0.0:8480</value>
    </property>
    <property>
        <name>dfs.journalnode.rpc-address</name>
        <value>0.0.0.0:8485</value>
    </property>
    <property>
        <name>ha.zookeeper.quorum</name>
        <value>Test-27:2181</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir.perm</name>
        <value>755</value>
    </property>
    <property>
        <name>dfs.block.local-path-access.user</name>
        <value>hadoop,root</value>
    </property>
    <property>
　　    <!--dfs.hosts.exclude定义的文件内容为,每个需要下线的机器，一行一个-->
        <name>dfs.hosts.exclude</name>
　      <value>/usr/local/hadoop/etc/hadoop/excludes</value>
    </property>
</configuration>

```

###### mapred-site.xml

``` sh
[hadoop@hdfs-master hadoop]$ cat mapred-site.xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.address</name>
        <value>nmaster:10020</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.webapp.address</name>
        <value>nmaster:19888</value>
    </property>
</configuration>

```

###### yarn-site.xml

``` sh
[hadoop@hdfs-master hadoop]$ cat yarn-site.xml
<?xml version="1.0"?>
<configuration>
    <property>
        <name>yarn.resourcemanager.connect.retry-interval.ms</name>
        <value>2000</value>
    </property>
    <property>
        <name>yarn.resourcemanager.ha.enabled</name>
        <value>true</value>
    </property>
    <property>
        <name>yarn.resourcemanager.ha.rm-ids</name>
        <value>rm1,rm2</value>
    </property>
    <property>
        <name>ha.zookeeper.quorum</name>
        <value>hdfs-master:2181</value>
    </property>

    <property>
        <name>yarn.resourcemanager.ha.automatic-failover.enabled</name>
        <value>true</value>
    </property>
    <property>
        <name>yarn.resourcemanager.hostname.rm1</name>
        <value>hdfs-master</value>
    </property>

    <property>
        <name>yarn.resourcemanager.hostname.rm2</name>
        <value>hdfs-slave</value>
    </property>
    <!--在hdfs-master上配置rm1,在hdfs-slave上配置rm2,注意：一般都喜欢把配置好的文件远程复制到其它机器上，但这个在YARN的另一个机器上一定要修改 -->
    <property>
        <name>yarn.resourcemanager.ha.id</name>
        <value>rm1</value>
    </property>
    <!--开启自动恢复功能 -->
    <property>
        <name>yarn.resourcemanager.recovery.enabled</name>
        <value>true</value>
    </property>
    <!--配置与zookeeper的连接地址 -->
    <property>
        <name>yarn.resourcemanager.zk-state-store.address</name>
        <value>hdfs-master:2181</value>
    </property>
    <property>
        <name>yarn.resourcemanager.store.class</name>
        <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>
    </property>
    <property>
        <name>yarn.resourcemanager.zk-address</name>
        <value>hdfs-master:2181</value>
    </property>
    <property>
        <name>yarn.resourcemanager.cluster-id</name>
        <value>hadoop-yarn</value>
    </property>
    <!--schelduler失联等待连接时间 -->
    <property>
        <name>yarn.app.mapreduce.am.scheduler.connection.wait.interval-ms</name>
        <value>5000</value>
    </property>
    <!--配置rm1 -->
    <property>
        <name>yarn.resourcemanager.address.rm1</name>
        <value>hdfs-master:8132</value>
    </property>
    <property>
        <name>yarn.resourcemanager.scheduler.address.rm1</name>
        <value>hdfs-master:8130</value>
    </property>
    <property>
        <name>yarn.resourcemanager.webapp.address.rm1</name>
        <value>hdfs-master:8188</value>
    </property>
    <property>
        <name>yarn.resourcemanager.resource-tracker.address.rm1</name>
        <value>hdfs-master:8131</value>
    </property>
    <property>
        <name>yarn.resourcemanager.admin.address.rm1</name>
        <value>hdfs-master:8033</value>
    </property>
    <property>
        <name>yarn.resourcemanager.ha.admin.address.rm1</name>
        <value>hdfs-master:23142</value>
    </property>
    <!--配置rm2 -->
    <property>
        <name>yarn.resourcemanager.address.rm2</name>
        <value>hdfs-slave:8132</value>
    </property>
    <property>
        <name>yarn.resourcemanager.scheduler.address.rm2</name>
        <value>hdfs-slave:8130</value>
    </property>
    <property>
        <name>yarn.resourcemanager.webapp.address.rm2</name>
        <value>hdfs-slave:8188</value>
    </property>
    <property>
        <name>yarn.resourcemanager.resource-tracker.address.rm2</name>
        <value>hdfs-slave:8131</value>
    </property>
    <property>
        <name>yarn.resourcemanager.admin.address.rm2</name>
        <value>hdfs-slave:8033</value>
    </property>
    <property>
        <name>yarn.resourcemanager.ha.admin.address.rm2</name>
        <value>hdfs-slave:23142</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
        <value>org.apache.hadoop.mapred.ShuffleHandler</value>
    </property>
    <property>
        <name>yarn.nodemanager.local-dirs</name>
        <value>/data/hadoop/data/yarn/local</value>
    </property>
    <property>
        <name>yarn.nodemanager.log-dirs</name>
        <value>/data/hadoop/log/yarn</value>
    </property>
    <property>
        <name>mapreduce.shuffle.port</name>
        <value>23080</value>
    </property>
    <!--故障处理类 -->
    <property>
        <name>yarn.client.failover-proxy-provider</name>
        <value>org.apache.hadoop.yarn.client.ConfiguredRMFailoverProxyProvider</value>
    </property>
    <property>
        <name>yarn.resourcemanager.ha.automatic-failover.zk-base-path</name>
        <value>/yarn-leader-election</value>
    </property>
</configuration>

```



##### 同步配置文件
- 同步到所有节点

``` sh
[hadoop@hdfs-master hadoop]$ scp -r /usr/local/hadoop/etc/hadoop hdfs-slave:/usr/local/hadoop/etc/

```
##### 修改 slave的ha.id

``` sh
[hadoop@hdfs-slave hadoop]$ cat yarn-site.xml |grep -a2 ha.id
    <!--在hdfs-master上配置rm1,在hdfs-slave上配置rm2,注意：一般都喜欢把配置好的文件远程复制到其它机器上，但这个在YARN的另一个机器上一定要修改 -->
    <property>
        <name>yarn.resourcemanager.ha.id</name>
        <value>rm2</value>
    </property>

```

##### 启用Hadoop

- 所有节点执行

``` sh
./sbin/hadoop-daemon.sh start journalnode
```

- master 

``` sh
[hadoop@hdfs-master hadoop]$ ./bin/hadoop namenode -format
[hadoop@hdfs-master hadoop]$ ./bin/hdfs zkfc -formatZK
[hadoop@hdfs-master hadoop]$ ./sbin/start-dfs.sh 
[hadoop@hdfs-master hadoop]$ ./sbin/start-yarn.sh
[hadoop@hdfs-master hadoop]$ jps
16896 Jps
30194 NodeManager
27363 JournalNode
30083 ResourceManager
29925 DFSZKFailoverController
29590 DataNode
29448 NameNode
``` 

- slave 

``` sh
查看此时的slave 节点进程:
[hadoop@hdfs-slave hadoop]$ jps
28489 NodeManager
32345 Jps
25675 JournalNode
28380 DFSZKFailoverController
28222 DataNode

## 同步master节点源数据
[hadoop@hdfs-slave hadoop]$ ./bin/hdfs namenode -bootstrapStandby
...........
=====================================================
About to bootstrap Standby ID nslave from:
           Nameservice ID: hadoop-node
        Other Namenode ID: nmaster
  Other NN's HTTP address: http://hdfs-master:50070
  Other NN's IPC  address: hdfs-master/192.168.3.27:9000
             Namespace ID: 910041107
            Block pool ID: BP-847688455-192.168.3.27-1516010305437
               Cluster ID: CID-15090545-231e-4ba6-a593-1afc74857e9c
           Layout version: -63
       isUpgradeFinalized: true
=====================================================
..... common.Storage: Storage directory /data/hadoop/data/dfs/name has been successfully formatted.

[hadoop@hdfs-slave hadoop]$ ./sbin/hadoop-daemon.sh start namenode

[hadoop@hdfs-slave hadoop]$ ./sbin/yarn-daemon.sh start resourcemanager
[hadoop@hdfs-slave hadoop]$ jps
28489 NodeManager
857 Jps
25675 JournalNode
28380 DFSZKFailoverController
32765 NameNode
28222 DataNode
478 ResourceManager


```

##### web-ui
- http://hdfs-master:50070/


##### 管理命令
- 状态查看

``` sh

[hadoop@hdfs-master hadoop]$ ./bin/hdfs haadmin -getServiceState nmaster
active
[hadoop@hdfs-master hadoop]$ ./bin/hdfs haadmin -getServiceState nslave
standby

```

- 未配置自动切换手动操作

``` sh

这条命令的意思是，nslave将变成standby，nmaster变成active。而且手动状态下需要重启服务
[hadoop@hdfs-master hadoop]$ ./bin/hdfs haadmin -failover --forcefence --forceactive nmaster nslave

[hadoop@hdfs-master hadoop]$ ./bin/hdfs haadmin -transitionToActive nslave
[hadoop@hdfs-master hadoop]$ ./bin/hdfs haadmin -transitionToStandby nmaster

```
