---
layout: post
categories: ELK
title: Elasticsearch 5.x 集群配置
date: 2017-01-23 13:59:06 +0800
description: Elasticsearch 5.x 集群配置
keywords: ELK elasticsearch
---


![这里写图片描述](http://img.blog.csdn.net/20170116142915212?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


## Elastic Stack 套件简介
* 详情请查看：https://www.elastic.co/products
![这里写图片描述](http://img.blog.csdn.net/20170118173626845?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### Elasticsearch
* 是Elastic Stack中最核心的组件，它是分布式、支持RESTful搜索和分析的引擎；并集中存储了你的数据；

### Logstash
* 数据处理的pipeline（INPUTS->FILTERS->OUTPUTS;
![这里写图片描述](http://img.blog.csdn.net/20170118173544284?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### Kibana
* 可视化你的Elasticsearch数据；
![这里写图片描述](http://img.blog.csdn.net/20170118204528388?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### X-Pack
* 对Elastic Stack提供了安全、警报、监控、报表、图表于一身的扩展包；
* 需付费 [X-Pack License](https://www.elastic.co/subscriptions)（可注册1年免费 License）
* 虽然x-pack被设计为一个无缝的工作，但是你可以轻松的启用或者关闭一些功能;


### Beats (本文未使用)
* 传输数据平台，作为一个轻量级代理从成千上万台机器上发送数据给Logstash或Elasticsearch；
* 按照类型细化成四类：FILEBEAT(Log Files)、METRICBEAT(Metrics)、PACKETBEAT(Network Data）、WINLOGBEAT(Windows Event Logs)。
* Beats是一系列叫XXXbeat的小工具，最常用的是Filebeat。原先属于Logstash的一个模块，后来单独拆分，根据不同的功能衍生了几个分支。
* Filebeat是个很轻量级的工具，主要干这么三件事：
    * 监听本机指定日志文件的新增条目，类似 `tail -f`

    * 做最原始的过滤和处理

    * 发送到Logstash，或者Elastic Search，又或者其他地方
    
### Elastcisearch + logstash + kibana + x-pack    


----


## 踩坑记录：
* 1. 直接复制2.X的elasticsearch.yml配置文件会有大量报错

    * 1）"node settings must not contain any index level settings" 不支持索引级别设置

    * 2）不支持脚本设置script。

    * 3）bootstrap.mlockall 改为了 bootstrap.memory_lock。
    
    * 所以直接在5.0的elasticsearch.yml进行参数设定。
    * ./config/ 里面新增了 jvm.options 和 log4j2.properties，取消了logging.yml。
    * 内存的配置可以直接在 jvm.options 里面。

* 2. 启动报错，提示调高JVM线程数

``` bash
bootstrap checks failed
max virtual memory areas vm.max_map_count [65530] likely too low, increase to at least [262144]


解决方法：
vi /etc/sysctl.conf
vm.max_map_count=262144 

```

* 3. 启动报错 bootstrap.memory_lock: true  开启此项报错

``` bash
解决方法：
# tail -n 2 /etc/security/limits.conf 
es soft memlock unlimited   
es hard memlock unlimited

```
-----

## ELK工作原理图
![这里写图片描述](http://img.blog.csdn.net/20170118173717955?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



## 环境准备：

### 1. 配置host文件
* 准备4台主机 
    * es-0 主节点和数据节点
    * es-1 主节点和数据节点
    * es-2 主节点和数据节点
    * es-3 数据节点


``` bash
[root@es-0 ~]# cat /etc/hosts
....

192.168.1.10 es-0    
192.168.1.11 es-1 
192.168.1.12 es-2
192.168.1.13 es-3

```

### 2. 配置SSH 双机互信

``` bash
此处将通过es-0 节点直接切换到其他节点进行操作免去输入密码
[root@es-0 ~]# ssh-keygen -t rsa [一路回车]
[root@es-0 ~]# cd .ssh && cat id_rsa.pub > authorized_keys 
[root@es-0 ~]# chmod 0600 .ssh/authorized_keys
[root@es-0 ~]# ssh-copy-id -i .ssh/id_rsa.pub root@es-1
[root@es-0 ~]# ssh-copy-id -i .ssh/id_rsa.pub root@es-2
[root@es-0 ~]# ssh-copy-id -i .ssh/id_rsa.pub root@es-3

```

### 3. 添加文件同步脚本
* [脚本文件查看此处](http://blog.csdn.net/zxf_668899/article/details/54314898)

``` bash
[root@es-0 ~]# tree /home/maintenance/
/home/maintenance/
├── config
│   └── config.sh
├── docmdoncluster.sh
└── updatefiletocluster.sh

1 directory, 3 files
[root@es-0 ~]# vim /etc/profile
export PATH=/home/maintenance:$PATH
[root@es-0 ~]# vim /home/maintenance/config/config.sh
#!/bin/bash
file_ips="es-1 es-2 es-3"
command_ips="localhost $file_ips"
port=22

``` 

### 4. 同步host文件

``` bash
[root@es-0 ~]# updatefiletocluster.sh /etc/hosts
file dir is /etc


==================starting rsync file /etc/hosts to es-1=====================
remote dir is existed!
sending incremental file list
host

sent 76 bytes  received 31 bytes  214.00 bytes/sec
total size is 4  speedup is 0.04


==================starting rsync file /etc/hosts to es-2=====================
remote dir is existed!
sending incremental file list
host

sent 76 bytes  received 31 bytes  214.00 bytes/sec
total size is 4  speedup is 0.04


==================starting rsync file /etc/hosts to es-3=====================
remote dir is existed!
sending incremental file list
host

sent 76 bytes  received 31 bytes  214.00 bytes/sec
total size is 4  speedup is 0.04
*****************finished!!!*******************

```

### 5. 配置java环境：

``` bash
[root@es-0 ~]# ls /opt/
jdk1.8.0_74
[root@es-0 ~]# ln -s /opt/jdk1.8.0_74 /opt/jdk
添加变量：
[root@es-0 ~]# vim /etc/profile  
export JAVA_HOME=/opt/jdk
export PATH=${JAVA_HOME}/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/jre/lib/dt.jar:$JAVA_HOME/lib/tools.jar

```

### 6. 系统及内核参数设置：

``` sh
[root@es-0 ~]# cat sys_init.sh
# /etc/security/limits.conf
echo -e "* soft nproc unlimited" >> /etc/security/limits.conf
echo -e "* hard nproc unlimited" >> /etc/security/limits.conf
echo -e "* soft nofile 655350" >> /etc/security/limits.conf
echo -e "* hard nofile 655350" >> /etc/security/limits.conf

# /etc/proflie
echo -e "ulimit -SHn 655350" >> /etc/profile
echo -e "ulimit -SHu unlimited" >> /etc/profile
echo -e "ulimit -SHd unlimited" >> /etc/profile
echo -e "ulimit -SHm unlimited" >> /etc/profile
echo -e "ulimit -SHs unlimited" >> /etc/profile
echo -e "ulimit -SHt unlimited" >> /etc/profile
echo -e "ulimit -SHv unlimited" >> /etc/profile

# /etc/sysctl.conf   (踩坑记录)
echo -e "vm.max_map_count=262144" >> /etc/sysctl.conf

[root@es-0 ~]# source /etc/profile
[root@es-0 ~]# ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 126606
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 655350
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) unlimited
cpu time               (seconds, -t) unlimited
max user processes              (-u) unlimited
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited

```

### 7. 开启 bootstrap.memory_lock: true 

``` sh  
[root@es-0 ~]# tail -n 2 /etc/security/limits.conf    (踩坑记录)
es soft memlock unlimited   
es hard memlock unlimited

注： es 为用户名，稍后会创建es用户

```




## 安装包下载
* https://www.elastic.co/products
    * elasticsearch-5.1.1.tar.gz
    * kibana-5.1.1-linux-x86_64.tar.gz
    * logstash-5.1.1.tar.gz
    * x-pack-5.1.1.zip
  

    
## 配置 esticsearch 5.1
### 1. 用户及目录权限设置
* 安装包均解压至 `/opt/source` 目录下并 `link` 到/opt/下

``` sh
[root@es-0 ~]# useradd es
[root@es-0 ~]# chown -R es:es /opt/source/
```


### elasticsearch 使用es用户操作！！！
###  2. 配置文件 
* [参数配置详解](http://blog.csdn.net/zxf_668899/article/details/54582849)
* es-0 配置文件

``` sh
[es@es-0 ~]$ grep -v "^#" /opt/es/config/elasticsearch.yml 
cluster.name: test-elk 
node.name: node-0   [需修改]
node.master: true   [数据节点需修改为false]
node.data: true
path.data: /home/es/es-data
path.logs: /home/es/logs
bootstrap.memory_lock: true 
network.publish_host: es-0 [需修改]
network.bind_host: es-0    [需修改]
discovery.zen.minimum_master_nodes: 3
discovery.zen.ping.unicast.hosts: ["es-0", "es-1", "es-2", "es-3"]
discovery.zen.fd.ping_timeout: 120s
discovery.zen.fd.ping_retries: 6
discovery.zen.fd.ping_interval: 30s
client.transport.ping_timeout: 60s
discovery.zen.ping_timeout: 120s
xpack.security.enabled: false #关闭es认证 与kibana对应

```

* 同步配置文件到其他节点

``` sh
[es@es-0 ~]$ updatefiletocluster.sh /opt/es/config/elasticsearch.yml
```

* `注`：各节点只需要修改上面所标记 `需修改` 的参数项即可

### 3. 安装 x-pack
* 安装在所有节点

``` sh
[es@es-0 ~]$ cd /opt/es/ && ./bin/elasticsearch-plugin install file:///opt/source/x-pack-5.1.1.zip
```


### 4. 启动 elasticsearch并验证
* 各节点分别执行启动命令

``` sh
[es@es-0 ~]$ cd /opt/es/ && ./bin/elasticsearch -d  [后台进程启动]
$ curl http://es-0:9200/_cat/nodes
192.168.1.11 29 93 0 0.21 0.36 0.40 mdi - node-1
192.168.1.10 70 95 1 0.28 0.37 0.40 mdi * node-0
192.168.1.12 61 93 1 0.25 0.31 0.37 mdi - node-2
192.168.1.13 34 97 0 0.57 0.46 0.42 di  - node-3
[es@es-0 ~]$ curl http://es-0:9200/_cat/health?v 

[es@es-0 ~]$ curl http://es-0:9200/_cat/plugins
node-3 x-pack 5.1.1
node-1 x-pack 5.1.1
node-0 x-pack 5.1.1
node-2 x-pack 5.1.1  

```

* 注： 禁用了认证功能，如果启用了认证,访问时需要指定用户名密码

``` sh
curl -u elastic:changeme http://es-0:9200/_cat/nodes
```


## logstash 
### 配置文件

``` sh
[root@es-3 ~]# cat /opt/logstash/conf/nginx.conf 
input {
    file {
        path => "/tmp/nginx/access*.log"
        start_position => beginning
    }
}
filter {
    grok {
        match => { "message" => "%{COMBINEDAPACHELOG} %{QS:x_forwarded_for}"}
    }
    date {
        match => [ "timestamp", "UNIX" ]
    }
    geoip {
        source => "clientip"
    }
}
output {
    elasticsearch { 
    hosts => "es-0:9200" 
    index => "nginx-%{+YYYY.MM.dd}"
    }
    stdout { codec => rubydebug }
}

```

*  注： 如果elasticsearch 启用了x-pack的认证功能则需要添加用户名密码

``` sh
.....
output {
    elasticsearch { 
    hosts => "es-0:9200" 
    index => "nginx-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "changeme"
    }
 ....
}
```

### 启动
``` sh
[root@es-3 ~]# cd /opt/logstash && nohup ./bin/logstash -f config/nginx -l logs/nginx.log &
```

## kibana
### 1. 配置文件

``` sh
[root@es-0 ~]# egrep -v "^#|^$" /opt/kibana/config/kibana.yml 
server.host: "kibana"
server.name: "test-kibana"
elasticsearch.url: "http://es-0:9200"
logging.dest: ./kibana.log
xpack.security.enabled: false

```

### 2. 安装 x-pack

``` sh
[root@es-0 ~]# cd /opt/kibana && ./bin/kibana-plugin install file:///opt/source/x-pack-5.1.1.zip
```

### 3. 启动kibana

``` sh
[root@es-0 ~]# cd /opt/kibana && nohup ./bin/kibana &
```

### 4. 页面访问
* 通过 http://es-0:5601 访问kibana 


![这里写图片描述](http://img.blog.csdn.net/20170116140039073?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![这里写图片描述](http://img.blog.csdn.net/20170116140052433?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


## x-pack 安全认证  `文本示例未开启安全认证功能`
* 如果开启安全认证需要在kibana.yml进行相关设置

``` sh
elasticsearch.username: "user"
elasticsearch.password: "password"
注： 默认kibana会使用初始用户名密码连接 elasticsearch
```

### license 注册信息
* [license Updating Your License](https://www.elastic.co/guide/en/x-pack/current/installing-license.html) 官方文档
* 注册获取基础 [license](https://register.elastic.co/xpack_register) 有效期 1 年

``` sh
curl -XPUT -u elastic:password 'http://<host>:<port>/_xpack/license' -d @license.json
@license.json 申请得到的json文件，复制文件中的所有内容，粘贴在此。

如果提示需要acknowledge，则设置为true
curl -XPUT -u elastic:password 'http://<host>:<port>/_xpack/license?acknowledge=true' -d @license.json

查看安装结果信息
curl -XGET -u elastic:password 'http://<host>:<port>/_license'

不同版本功能
https://www.elastic.co/subscriptions
```

### x-pack 用户管理
* x-pack安装之后有一个超级用户 `elastic` 默认密码 `changeme` 前面有一再的说明；
* 可以使用该用户创建和修改其他用户，也可以通过kibana的web界面进行用户和用户组的管理；

``` sh
修改elastic用户的密码：
$ curl -XPUT -u elastic 'localhost:9200/_xpack/security/user/elastic/_password' -d '{
  "password" : "123456"
}'


修改kibana用户的密码：
$ curl -XPUT -u elastic 'localhost:9200/_xpack/security/user/kibana/_password' -d '{
  "password" : "123456"
}'


创建用户组和角色，创建所属用户
创建 `admin` 用户组，该用户组对 `nginx*` 有all权限，对.kibana*有manage，read，index权限
$ curl -XPOST -u elastic 'localhost:9200/_xpack/security/role/admin' -d '{
  "indices" : [
    {
      "names" : [ "nginx*" ],
      "privileges" : [ "all" ]
    },
    {
      "names" : [ ".kibana*" ],
      "privileges" : [ "manage", "read", "index" ]
    }
  ]
}'


创建 test 用户，密码是 passwd
$ curl -XPOST -u elastic 'localhost:9200/_xpack/security/user/test' -d '{
  "password" : "passwd",
  "full_name" : "test",
  "email" : "XXX@XXX.com",
  "roles" : [ "admin" ]
}'
```

### 卸载 x-pack

``` sh
./bin/elasticsearch-plugin remove x-pack
./bin/kibana-plugini remove x-pack
```


* [官方文档](https://www.elastic.co/guide/index.html) 
* [elasticsearch 权威指南](http://www.learnes.net/) 
* [ELK stack 权威指南](http://kibana.logstash.es/content/logstash/) 
* [ELK 开发指南](https://endymecy.gitbooks.io/elasticsearch-guide-chinese/content/index.html) 


转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
