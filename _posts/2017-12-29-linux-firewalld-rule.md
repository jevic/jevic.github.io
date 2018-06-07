---
layout: post
categories: Linux
title: Linux Firewalld 自定义富规则配置(三)
date: 2016-11-09 13:01:35 +0800
description: firewalld
keywords: linux firewalld iptables
---

## 详细的说明自行参考 
``` sh
firewall-cmd --help
```

* 传递的参数 <args> 与 iptables, ip6tables 以及 ebtables 一致！
Centos7 Firewall 用户操作接口依然调用系统内核的`iptables`模块来设定规则！

* 直接选项 `–direct` 需要是直接选项的第一个参数。
将命令传递给防火墙。参数 <args> 可以是 iptables, ip6tables 以及 ebtables 命令行参数。
 ```
 firewall-cmd --direct --passthrough { ipv4 | ipv6 | eb } <args>
 ```

* 为表 `<table>` 增加一个新链 `<chain> `。

```
firewall-cmd --direct --add-chain { ipv4 | ipv6 | eb } <table> <chain>
```

* 从表` <table>` 中删除链 `<chain> `。

```
firewall-cmd --direct --remove-chain { ipv4 | ipv6 | eb } <table> <chain>
```

* 查询 `<chain>` 链是否存在与表 `<table >`. 如果是，返回0,否则返回1.

```
firewall-cmd --direct --query-chain { ipv4 | ipv6 | eb } <table> <chain>
```

* 获取用空格分隔的表 `<table>` 中链的列表。

```
firewall-cmd --direct --get-chains { ipv4 | ipv6 | eb } <table>
```

* 为表 `<table>` 增加一条参数为 `<args>` 的链 `<chain>` ，优先级设定为 `<priority>`。

```
firewall-cmd --direct --add-rule { ipv4 | ipv6 | eb } <table> <chain> <priority> <args>
```

## 参考示例：

* 方式-：[reload 生效，修改后重启才可生效]

```
# firewall-cmd --permanent --direct --add-rule ipv4 filter INPUT 1 -s 192.168.1.0/24 -p tcp --dport=22 -j ACCEPT
# firewall-cmd --permanent --direct --add-rule ipv4 filter INPUT 2 -p tcp --dport=22 -j DROP
```

* 方式二：[重载或重启后失效]

``` sh
# iptables -t filter -I INPUT_direct -s 192.168.1.20 -p tcp --dport=22 -j ACCEPT
# iptables -A INPUT_direct -p tcp --dport=22 -j DROP 
```

- 从表 `<table>` 中删除带参数 `<args>` 的链 `<chain>`。

``` sh
# firewall-cmd --direct --remove-rule { ipv4 | ipv6 | eb } <table> <chain> <args>
# firewall-cmd --direct --remove-rule ipv4 filter INPUT 2 -p tcp --dport=22 -j DROP
```

* 查询带参数 `<args>` 的链 `<chain>` 是否存在表 `<table>` 中

```
firewall-cmd --direct --query-rule { ipv4 | ipv6 | eb } <table> <chain> <args>
```

获取表 `<table>` 中所有增加到链 `<chain>` 的规则，并用换行分隔。

```
firewall-cmd --direct --get-rules { ipv4 | ipv6 | eb } <table> <chain>
```

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
