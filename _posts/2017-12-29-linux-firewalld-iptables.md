---
layout: post
categories: Linux
title: Linux Firewalld 防火墙配置(一)
date: 2016-07-23 14:48:08 +0800
description: linux iptables
keywords: linux iptables firewalld
---

# firewalld 常用配置

## 永久设定
`--permanent`  当设定永久状态时 在命令开头或者结尾处加入此参数,否则重载或重启防火墙后设置失效！

## 开放端口
```
# firewall-cmd --zone=public --add-port=80/tcp --permanent
# firewall-cmd --zone=public --add-port=22/tcp --permanent

可以一次指定多个：
# firewall-cmd --zone=public  --permanent --add-port=111/tcp --add-port=139/tcp --add-port=445/tcp

```
## 加载配置
```
firewall-cmd --reload
```

## 查看端口
```
# firewall-cmd --list-port
# firewall-cmd --zone=public --list-ports
```

## 开启伪装
```
# firewall-cmd [--zone=zone] --add-masquerade
# firewall-cmd --remove-masquerade
# firewall-cmd --query-masquerade
```
## 添加区域接口
```
# firewall-cmd [--zone=zone] --add-interface=<interface>
# firewall-cmd --zone=public --add-interface=eth0

列出全部启用的区域的特性
firewall-cmd --list-all-zones

输出区域 <zone> 全部启用的特性。如果省略区域，将显示默认区域的信息
# firewall-cmd --zone=public --list-all
```

## 启用服务
```
firewall-cmd --add-service=http
firewall-cmd --add-service=vnc-server
firewall-cmd --zone=public --add-service=nfs --add-service=samba --add-service=samba-client --permanent

firewall-cmd --remove-service=service   移除服务

查询：firewall-cmd --list-service
```

## NAT地址转换
```
firewall-cmd [--zone=<zone>] --add-forward-port=port=<port>[-<port>]:proto=<protocol> { :toport=<port>[-<port>] | :toaddr=<address> | :toport=<port>[-<port>]:toaddr=<address> }
```
### IP端口转发
```
# firewall-cmd --add-forward-port=222:proto=tcp:toport=333:toaddr=192.168.1.100
```
### 本地转发：
```
# firewall-cmd --add-forward-port=port=9898:proto=tcp:toport=8088:toaddr=
```
### 查询：
```
# firewall-cmd --list-forward-port
# firewall-cmd --list-port
# firewall-cmd --list-all
```
### 移除：
```
# firewall-cmd --remove-forward-port=port=222:proto=tcp:toport=333:toaddr=
# firewall-cmd --remove-forward-port=222:proto=tcp:toport=333:toaddr=192.168.1.100
```
## 图形化配置工具
```
# firewall-config
```
## 自定义规则
```
/sbin/iptables -t filter -I INPUT_direct 2 -s 192.168.1.1 -p tcp --dport=22 -j DROP 
usage: --direct --add-rule { ipv4 | ipv6 | eb } <table> <chain> <priority> <args>

# firewall-cmd --permanent --direct --add-rule ipv4 filter INPUT 1 -s 192.168.1.0/24 -p tcp --dport=22 -j ACCEPT
# firewall-cmd --permanent --direct --add-rule ipv4 filter INPUT 2 -p tcp --dport=22 -j DROP
# firewall-cmd --reload
# firewall-cmd --direct --get-all-rules
ipv4 filter INPUT 1 -s 192.168.1.0/24 -p tcp --dport=22 -j ACCEPT
ipv4 filter INPUT 2 -p tcp --dport=22 -j DROP

```

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
