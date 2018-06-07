---
layout: post
categories: Linux
title: Linux firewalld xml配置文件(二)
date: 2016-10-19 12:55:18 +0800
description: iptables
keywords: iptables firewalld linux
---

- 在 CentOS 6 防火墙 iptables 是以 Rules 写在 Chain 建立规则然后给 kernel 去执行，每次重建规则必须先清除原有规则，这会导致既有连线中断。
- 到 CentOS 7 之后改用 firewalld 以 zone 的区域分割观念来建立，并以动态设定方式执行避免中断的问题。
- 请注意：不能同时执行 iptables 跟 firewalld 这会造成冲突错误。


## Firewalld 相关路径
- /etc/firewalld：设定档路径
- /usr/bin/：firewall-cmd 指令所在的位置
- /usr/lib/firewalld/：firewalld 预设的设定资料 (xml 格式)，例如 nfs.xml 就定义了 `<port protocol="tcp" port="2049"/>`

-------------

* 所谓的 zone 就表示主机位於何种环境，需要设定哪些规则，在 firewalld 里共有 7 个zones
> 先决定主机要设定在那个区域 zone >> 再往该 zone 设定规则 >>> 重新读取设定档 sudo firewall-cmd --reload

* public： 公开的场所，不信任网域内所有连线，只有被允许的连线才能进入，一般只要设定这里就可以了
For use in public areas. You do not trust the other computers on the network to not harm your computer. Only selected incoming connections are accepted.

* external： 公开的场所，应用在IP是NAT的网路 
For use on external networks with masquerading enabled especially for routers. You do not trust the other computers on the network to not harm your computer. Only selected incoming connections are accepted.

* dmz： (Demilitarized Zone) 非军事区，允许对外连线，内部网路只有允许的才可以连线进来 
For computers in your demilitarized zone that are publicly-accessible with limited access to your internal network. Only selected incoming connections are accepted.

* work： 公司、工作的环境 
For use in work areas. You mostly trust the other computers on networks to not harm your computer. Only selected incoming connections are accepted.

* home： 家庭环境 
For use in home areas. You mostly trust the other computers on networks to not harm your computer. Only selected incoming connections are accepted.

* internal： 内部网路，应用在NAT设定时的对内网路 
For use on internal networks. You mostly trust the other computers on the networks to not harm your computer. Only selected incoming connections are accepted.

* trusted： 接受所有的连线
All network connections are accepted.

* drop： 任何进入的封包全部丢弃，只有往外的连线是允许的 
Any incoming network packets are dropped, there is no reply. Only outgoing network connections are possible.

* block： 任何进入的封包全部拒绝，并以 icmp 回覆对方 ，只有往外的连线是允许的
Any incoming network connections are rejected with an icmp-host-prohibited message for IPv4 and icmp6-adm-prohibited for IPv6. Only network connections initiated from within the system are possible.

---------------------

* 预设主机是被放在 public zone 区域，并有开启两个服务 dhcpv6-client ssh
* 在这样的设定下，任何来源都可以透过 ssh 服务来连接到本主机，但其他的服务 service-port 都是关闭的。

##  显示目前的设定 
``` sh
# firewall-cmd --list-all
public (default, active)
interfaces: ens160
sources:
services: dhcpv6-client ssh
ports:
masquerade: no
forward-ports:
icmp-blocks:
rich rules:
```

* 若你的环境没在用 DHCP ，则可以将他关掉 DHCP 服务 port
* 关掉 DHCP 服务 port
```
# sudo firewall-cmd --zone=public --remove-service dhcpv6-client
```
* 暂时允许外部连接本机 DNS 服务
* 暂时开启 DNS port 53

``` sh
# sudo systemctl start named
# sudo systemctl enable named
# sudo firewall-cmd --add-service=dns
# sudo firewall-cmd --reload
# firewall-cmd --list-all
public (default, active)
interfaces: ens160
sources:
services: dhcpv6-client dns ssh
ports: (略)
```

* 永久开启 DNS port 53 

``` sh
# sudo firewall-cmd --add-service=dns --permanent
# sudo firewall-cmd --reload
```

-----

QA：为何可以直接指定加入的服务名称为dns 呢？
Ans：他会参考 /usr/lib/firewalld/services/ 下的服务，例如 dns.xml ，启用本服务就会连带的开启 TCP 跟 UDP 的 port 53

``` sh
<?xml version="1.0" encoding="utf-8"?>
<service>
<short>DNS</short>
<description>The Domain Name System (DNS) is used to provide and request host and domain names. Enable this option, if you plan
to provide a domain name service (e.g. with bind).</description>
<port protocol="tcp" port="53"/>
<port protocol="udp" port="53"/>
</service>
```

- 总共有这些服务

``` sh
# ls -1 /usr/lib/firewalld/services/
amanda-client.xml
amanda-k5-client.xml
bacula-client.xml
bacula.xml
ceph-mon.xml
ceph.xml
.....

```


## 修改主机的预设 zone 
* 前面说过预设为 public zone ，但有些主机服务是要建立在 DMZ 下，我们可以透过修改 /etc/firewalld/firewalld.conf 来将预设的区域改为 dmz

``` sh
# sudo vi /etc/firewalld/firewalld.conf
> 修改 DefaultZone=public
> 变成 DefaultZone=dmz
然后重新载入
# sudo firewall-cmd --reload
```


## 自行指定的连接端口 
```
# sudo firewall-cmd --add-port=8080/tcp --permanent
success
# sudo firewall-cmd --reload
success
# sudo firewall-cmd --list-all
public (default, active)
interfaces: ens160
sources:
services: dhcpv6-client dns ssh
ports: 8080/tcp
masquerade: no
```
---

QA：为何我设定暂时的 rules 都无效呢？
ANS：假设你临时要开放 port 8888，执行

``` sh
# sudo firewall-cmd --add-port=8888/tcp 
# sudo firewall-cmd --reload
# sudo firewall-cmd --list-all

却发现 --list-all 的结果都没将 port:8888 这 rule 给加入
这是因为你在第二步骤时做了 reload ，不是设定没用而是被你的 reload 给清除了
# sudo firewall-cmd --add-port=8888/tcp
# sudo firewall-cmd --list-all
```

## 查看永久的设定
```
# firewall-cmd  --list-all --permanent
```


## 修改服务的预设连接端口 
假设你的主机web/www ，默认port 80 但有时候某些网站就不是使用 80 port ，这时候该如何修改呢？

``` sh
Step 1：先复制对应的 xml 档到 /etc/firewalld/services
# sudo cp /usr/lib/firewalld/services/http.xml  /etc/firewalld/services

Step 2：修改 http.xml
# sudo vi /etc/firewalld/services/http.xml

Step 3：修改对应连接端口
<port protocol="tcp" port="80"/>
改为
<port protocol="tcp" port="8080"/>

Step 4：重新载入 
# sudo firewall-cmd --reload

```


## 限制某服务只能从哪IP连入 

``` sh
通常我们会限制 ssh 只能从某些IP来进来，我们可以使用 rich-rule 来加入
# sudo firewall-cmd --add-rich-rule="rule family="ipv4" source address="192.168.1.88" service name="ssh" accept"
或 ip subnet
# sudo firewall-cmd --add-rich-rule="rule family="ipv4" source address="192.168.1.0/24" service name="ssh" accept"
```


## 限制某 port 只能从哪IP连入 
```
# sudo firewall-cmd --add-rich-rule="rule family="ipv4" source address="192.168.12.9" port port="8080" protocol="tcp" accept"
注意：port port="8080" 不是写错！这是他的格式
```

## 直接指定 rule 到 INPUT  chain
```
# sudo firewall-cmd --permanent --direct --add-rule ipv4 filter INPUT 0 -p tcp -s "192.168.12.9" --dport 22 -j ACCEPT
这样的写法使用 # firewall-cmd --list-all 是看不到的，要用 iptables -L -n
```


## 查看预设载入的 rule 
所有的 zone 设定档会放在 /etc/firewalld/zones 跟 /usr/lib/firewalld/zones/ ，
你所执行的 --permanent 参数会写在 /etc/firewalld/zones 对应的 zone 文件内（例：public.xml），所以当你

``` sh
# sudo firewall-cmd --add-service=dns --permanent
就会在 /etc/firewalld/zones/public.xml 变成 多了 <service name="dns"/>
<?xml version="1.0" encoding="utf-8"?>
  <zone>
  <short>Public</short>
  <description>For use in public areas. You do not trust the other computers on networks to not harm your computer. Only selected incoming connections are accepted.</description>
  <service name="dhcpv6-client"/>
  <service name="ssh"/>
  <service name="dns"/>
  <port protocol="tcp" port="8080"/>
</zone>
```


## 从 /etc/sysconfig/iptables 转为 firewalld 的 direct  

```
假设你原有的 /etc/sysconfig/iptables 有规则
-A INPUT -s 140.113.12.9 -j ACCEPT
-A INPUT -m state --state NEW -m udp -p udp -s 140.113.0.0/16 --dport 123 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp -s 140.114.88.0/24 --dport 161 -j ACCEPT
要转换到 firewalld 的 direct 规则
新增 /etc/firewalld/direct.xml  ，如果你之前有执行过 #sudo firewall-cmd --permanent --direct .... 则自动的产生

新增/修改 direct.xml 增加对应上面的 rules

# sudo vi /etc/firewalld/direct.xml
<?xml version="1.0" encoding="utf-8"?>
<direct>
   <rule priority="0" table="filter" ipv="ipv4" chain="INPUT">-p tcp -s 192.168.12.9 --dport 22 -j ACCEPT</rule>
   <rule priority="0" table="filter" ipv="ipv4" chain="INPUT">-s 140.113.12.9 -j ACCEPT</rule>
   <rule priority="0" table="filter" ipv="ipv4" chain="INPUT">-p udp -s 140.113.0.0/16 --dport 123 -j ACCEPT</rule>
   <rule priority="0" table="filter" ipv="ipv4" chain="INPUT">-p tcp -s 140.114.88.0/24 --dport 161 -j ACCEPT</rule>
</direct>
```

## 从 zone 移除某项服务 
```
# sudo firewall-cmd --zone=public --add-service=http --permanent 
# sudo firewall-cmd --zone=public --remove-service=http --permanent

# sudo firewall-cmd --zone=public --add-port=8080/tcp --permanent 
# sudo firewall-cmd --zone=public --remove-port=8080/tcp --permanent
```


* port forward 将从某 port number 的封包转送另外的 port 或其他主机

```
# 将 80 port 转往 port 8080
# sudo firewall-cmd --zone="public" --add-forward-port=port=80:proto=tcp:toport=8080

# 将 80 port 转往其他台主机的 port 8080
# sudo firewall-cmd --zone="public" --add-forward-port=port=80:proto=tcp:toport=8080:toaddr=140.113.1.1
```

## Reference:
- [1]  [Red_Hat_Enterprise_Linux](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Security_Guide/sec-Using_Firewalls.html "Red_Hat")
- [2]  [youtube](https://www.youtube.com/watch?v=7_XwTdZlqes "youtube")
- [3]  [Introduction to FirewallD on CentOS](https://www.linode.com/docs/security/firewalls/introduction-to-firewalld-on-centos "Introduction to FirewallD on CentOS")


转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
