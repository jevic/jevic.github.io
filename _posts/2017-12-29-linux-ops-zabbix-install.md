---
layout: post
categories: OPS
title: Zabbix 手动快速安装
date: 2016-11-29 16:33:27 +0800
description: Zabbix 手动快速安装
keywords: zabbix linux
---


>手动快速安装配置 Zabbix 环境

## 环境准备

``` sh
[root@Server ~]# yum install -y epel-release
[root@Server ~]# yum install -y php php-gd php-mysql php-bcmath php-mbstring php-xml curl curl-devel net-snmp net-snmp-devel perl-DBI
[root@Server ~]# yum install -y httpd 

ntpdate ntp.sjtu.edu.cn
```


* 如果已经有数据库服务则无需重复安装

* mysql

``` sh
wget http://ftp.kernel.yunfan.com/soft/yangjie/bigdata/mysql57-community-release-el7-9.noarch.rpm
rpm -ivh mysql57-community-release-el7-9.noarch.rpm
yum install mysql-server
```

* mariadb

``` sh
[root@Server ~]# yum install -y mariadb mariadb-server 
[root@Server ~]# systemctl start mariadb
[root@Server ~]# systemctl enable mariadb

```

* 安装服务端和客户端

``` sh
[root@Server ~]# yum install -y zabbix-server-mysql zabbix-web-mysql zabbix-agent
```

* 创建数据库：

``` bash
[root@dbServer ~]# mysql -uroot -p
MariaDB [(none)]> create database zabbix character set utf8 collate utf8_bin;
MariaDB [(none)]> grant all privileges on zabbix.* to 'zabbix'@'192.168.1.10' identified by 'zabbix';
MariaDB [(none)]> flush privileges;
MariaDB [(none)]> quit
```

## Server

``` bash
[root@Server ~]# vi /etc/zabbix/zabbix_server.conf
DBHost=Server
DBName=zabbix
DBUser=zabbix
DBPassword=zabbix

[root@Server ~]# vim /etc/httpd/conf.d/zabbix.conf
 <IfModule mod_php5.c>
        php_value max_execution_time 300
        php_value memory_limit 128M
        php_value post_max_size 16M
        php_value upload_max_filesize 2M
        php_value max_input_time 300
        php_value always_populate_raw_post_data -1
        php_value date.timezone Asia/Shanghai  ## 添加

[root@Server ~]# systemctl start zabbix-server && systemctl enable zabbix-server
[root@Server ~]# systemctl start zabbix-agent && systemctl enable zabbix-agent
[root@Server ~]# systemctl start httpd && systemctl enable httpd



设置字体消除乱码：(从Windows系统复制简体字体到此目录)
[root@Server ~]# cd /usr/share/zabbix/fonts/
[root@Server ~]# ls simshei.ttc 
[root@Server ~]# vim /usr/share/zabbix/include/defines.inc.php
define('ZBX_GRAPH_FONT_NAME',           'simhei'); // font file name

http://192.168.1.10/zabbix   访问链接安装

```

## agent

* centos 6.x:

``` bash
yum install -y http://repo.zabbix.com/zabbix/3.2/rhel/6/x86_64/zabbix-agent-3.2.4-1.el6.x86_64.rpm
```

``` sh
centos 7.2:
[root@node1 ~]# cat zabbix.repo 
[zabbix]
name=Zabbix Official Repository - $basearch
baseurl=http://repo.zabbix.com/zabbix/3.2/rhel/7/$basearch/
enabled=1
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX-A14FE591

[zabbix-non-supported]
name=Zabbix Official Repository non-supported - $basearch 
baseurl=http://repo.zabbix.com/non-supported/rhel/7/$basearch/
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX
gpgcheck=0
```


``` sh
[root@node1 ~]# vim /etc/zabbix/zabbix_agentd.conf

PidFile=/var/run/zabbix/zabbix_agentd.pid
LogFile=/var/log/zabbix/zabbix_agentd.log
LogFileSize=0
Server=192.168.1.10
ServerActive=192.168.1.10
Hostname=Zabbix server
Include=/etc/zabbix/zabbix_agentd.d/*.conf

```


