---
layout: post
categories: OPS
title: Zabbix Docker运行及报警监控
date: 2017-12-02 16:50:24 +0800
description: Zabbix Docker运行及报警监控
keywords: zabbix
---

## Zabbix 
使用官方Docker 镜像运行Zabbix 
[docker-compose](https://github.com/jevic/docker/tree/master/zabbix) 编排文件


#### Server 配置 

``` sh

[root@zabbix etc]# cat zabbix_server.conf
PidFile=/tmp/zabbix_server.pid
LogFile=/tmp/zabbix_server.log
DBHost=x.x.x.x
DBName=zabbix
DBUser=zabbix
DBPassword=zabbix_pwd
DBSocket=/tmp/mysql.sock
Include=/usr/local/zabbix/etc/zabbix_server.conf.d/*.conf
StartPollers=16
SNMPTrapperFile=/var/log/snmptrap/snmptrap.log
Timeout=4
AlertScriptsPath=/usr/local/zabbix/share/zabbix/alertscripts
ExternalScripts=/usr/local/zabbix/share/zabbix/externalscripts
LogSlowQueries=3000

```


## 报警设置
[报警接口应用](https://github.com/jevic/wxalarm)

![](http://ok6h8mla5.bkt.clouddn.com/zabbix01.png)
![](http://ok6h8mla5.bkt.clouddn.com/zabbix02.png)
![](http://ok6h8mla5.bkt.clouddn.com/zabbix03.png)
![](http://ok6h8mla5.bkt.clouddn.com/zabbix04.png)



```
1. 脚本参数:
{ALERT.SUBJECT}
{ALERT.MESSAGE}

2. 动作:
默认接收人: 
服务器:{HOSTNAME1}发生: {TRIGGER.NAME} 故障!
告警主机:{HOSTNAME1}
告警时间:{EVENT.DATE} {EVENT.TIME}
告警等级:{TRIGGER.SEVERITY}
告警信息: {TRIGGER.NAME}
告警项目:{TRIGGER.KEY1}
问题详情:{ITEM.NAME}:{ITEM.VALUE}

服务器:{HOSTNAME1}: {TRIGGER.NAME}已恢复!
告警主机:{HOSTNAME1}
告警时间:{EVENT.DATE} {EVENT.TIME}
告警等级:{TRIGGER.SEVERITY}
告警信息: {TRIGGER.NAME}
告警项目:{TRIGGER.KEY1}
问题详情:{ITEM.NAME}:{ITEM.VALUE}


```

## 模板
- https://monitoringartist.github.io/zabbix-searcher/#
- 模板 -> 应用集 -> 监控项 -> 触发器

![](http://ok6h8mla5.bkt.clouddn.com/zabbix-temlpate-01.png)
![](http://ok6h8mla5.bkt.clouddn.com/zabbix-template-02.png)
![](http://ok6h8mla5.bkt.clouddn.com/zabbix-template-03.png)
![](http://ok6h8mla5.bkt.clouddn.com/zabbix-template-04.png)

### 触发器多参数示例
```
{Template  DT general  monitor for 302:ngx_status[count,499].last()}>10

```

## agent 配置文件
```
UserParameter=chk_timezone,if [ `date +%Z` != CST ];then echo 0 ;else echo 1;fi
UserParameter=tcp_status[*],/data/script/tcp_status.sh $1
UserParameter=chk_kt_rsync,/data/script/chk_kt_sync.sh
UserParameter=chk_openfile,/usr/sbin/lsof -n|wc -l
UserParameter=ngx_status[*],/data/script/ngx_status.sh $1 $2
UserParameter=chk_nginx_process,/bin/ps -ef|grep nginx|grep -v grep|wc -l
UserParameter=chk_io_usage,/usr/bin/iostat -x 1 2 | /usr/bin/perl -lane '$cnt++ and next if /Device/; $max < $F[-1] and $max = $F[-1] if $cnt > 1; END{ print $max || 0.00 }'
UserParameter=find_disk,/data/script/find_disk.sh
UserParameter=chk_disk_io[*],/data/script/chk_disk_io.sh $1 $2
UserParameter=chk_thread[*],/data/script/chk_thread.sh $1

```


