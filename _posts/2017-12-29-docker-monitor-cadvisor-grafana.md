---
layout: post
categories: Docker
title: Docker 监控应用 cAdvisor-InfluxDB-Grafana
date: 2017-01-23 11:29:45 +0800
description: cAdvisor-InfluxDB-Grafana-docker
keywords: docker
---



## 基础概念
### cAdvisor
​ **cAdvisor 为Docker容器用户提供了了解运行时容器资源使用和性能特征的工具。cAdvisor的容器抽象基于Google的lmctfy容器栈，因此原生支持Docker容器并能够“开箱即用”地支持其他的容器类型。cAdvisor部署为一个运行中的daemon，它会收集、聚集、处理并导出运行中容器的信息。这些信息能够包含容器级别的资源隔离参数、资源的历史使用状况、反映资源使用和网络统计数据完整历史状况的柱状图。**

### InfluxDB
 **InfluxDB 是一个开源分布式时序、事件和指标数据库。使用 Go 语言编写，无需外部依赖。其设计目标是实现分布式和水平伸缩扩展.**
 
* 其主要特色功能
    * 基于时间序列，支持与时间有关的相关函数（如最大，最小，求和等）
    * 可度量性：你可以实时对大量数据进行计算
    * 基于事件：它支持任意的事件数据

**​InfluxDB的主要特点**

*    无结构（无模式）：可以是任意数量的列可拓展的
* 支持min, max, sum, count, mean, median 等一系列函数，方便统计
* 原生的HTTP支持，内置HTTP API
* 强大的类SQL语法
* 自带管理界面，方便使用

### Grafana
 **Graphite 是一款开源的监控绘图工具。可以实时收集、存储、显示时间序列类型的数据（time series data），有些类似Kibana的东西。**

​- **以下是官方的说明**

* 用于可视化大型测量数据的开源程序，他提供了强大和优雅的方式去创建、共享、浏览数据。dashboard中显示了你不同metric数据源中的数据。
* 常用于因特网基础设施和应用分析，但在其他领域也有机会用到，比如：工业传感器、家庭自动化、过程控制等等。
* 有热插拔控制面板和可扩展的数据源，目前已经支持Graphite、Cloudwatch、Prometheus、InfluxDB、Elasticsearch。


## 镜像列表
``` sh
[root@master workdir]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
registry            latest              182810e6ba8c        36 hours ago        37.6 MB
grafana/grafana     4.0.2               757396810071        2 weeks ago         268.2 MB
jevic.io/nginx      alpine              d964ab5d0abe        4 weeks ago         54.89 MB
google/cadvisor     fejta               fe153dd2defc        5 weeks ago         736.7 MB
jevic.io/cadvisor   fejta               fe153dd2defc        5 weeks ago         736.7 MB
jevic.io/influxdb   0.13                39fa42a093e0        5 months ago        290.3 MB
tutum/influxdb      0.13                39fa42a093e0        5 months ago        290.3 MB
提示：请勿下载使用 influxdb:latest 镜像
```
## 启动脚本：
``` sh
[root@master workdir]# cat influxdb.sh 
#!/bin/bash
docker service create \
--network jevic-io \
-p 8083:8083 \
-p 8086:8086 \
--mount source=influxdb-vol,type=volume,target=/var/lib/influxdb \
--name=influxdb \
--constraint 'node.hostname==node01' \
jevic.io/influxdb:0.13

[root@master workdir]# cat cadvisor.sh 
#!/bin/bash
docker service create \
--network jevic-io \
--name cadvisor \
-p 8080:8080 \
--mode global \  #为每个节点创建一个服务,收集节点docker性能数据
--mount source=/var/run,type=bind,target=/var/run,readonly=false \
--mount source=/,type=bind,target=/rootfs,readonly=true \
--mount source=/sys,type=bind,target=/sys,readonly=true \
--mount source=/var/lib/docker,type=bind,target=/var/lib/docker,readonly=true \
jevic.io/cadvisor:fejta -storage_driver=influxdb -storage_driver_host=influxdb:8086 -storage_driver_db=cadvisor

[root@master workdir]# cat grafana.sh 
#!/bin/bash
docker service create \
--network jevic-io \
--name grafana \
#-e "GF_SECURITY_ADMIN_PASSWORD=passwd" \
--constraint 'node.hostname==master' \
-p 3000:3000 \
grafana/grafana:4.0.2

```

## influxdb 设置
* 访问 8083端口   用户名密码默认都为 root 
![这里写图片描述](http://img.blog.csdn.net/20161229184734547?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
* 创建数据库并查看
    * CREATE DATABASE "cadvisor"
    * SHOW DATABASES
    
![这里写图片描述](http://img.blog.csdn.net/20161229184824032?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![这里写图片描述](http://img.blog.csdn.net/20161229184837298?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
## cAdvisor 
* cdvisor运行以后，可以通过http://192.168.11.140:8080/ 查看到Docker运行的机器和容器状态
![这里写图片描述](http://img.blog.csdn.net/20161229190844435?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

* 通过http://192.168.11.140:8080/docker/，可以看到Docker服务器的基本信息，如Host、镜像数据、窗口数据等情况
![这里写图片描述](http://img.blog.csdn.net/20161229190854026?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## Grafana 图形配置
* 运行Grfana容器，通过浏览器打开http://192.168.11.130:3000，用户名admin，密码admin
* 配置数据源
![这里写图片描述](http://img.blog.csdn.net/20161229190133551?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
## 示例图【请右键打开新标签查看原图】
![这里写图片描述](http://img.blog.csdn.net/20161229190320600?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![这里写图片描述](http://img.blog.csdn.net/20161229190333616?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![这里写图片描述](http://img.blog.csdn.net/20161229190346804?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


## 参考链接
* http://docs.grafana.org
* https://www.brianchristner.io/how-to-setup-docker-monitoring/
* https://github.com/google/cadvisor
* https://github.com/vegasbrianc/docker-monitoring
* [五个Docker监控工具的对比](http://dockone.io/article/397)


转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
