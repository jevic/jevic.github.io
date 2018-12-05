---
layout: post
categories: Linux
title: Linux 高并发量系统内核优化参数
date: 2017-10-09 13:01:35 +0800
description: firewalld
keywords: linux
---

>针对高并发量系统内核优化参数 /etc/sysctl.conf

>根据实际场景调整相关参数


- 并发连接查看

```
# netstat -n | awk '/^tcp/ {++S[$NF]} END {for (a in S) print a, S[a]}'

# ss -s 

```



### 场景一 1分钟并发量超过10W+ 
> 此时不仅需要调整相关内核参数，也需要与业务及开发人员沟通调整使用方式以及代码方面的优化

```
net.ipv4.ip_forward = 0
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.default.accept_source_route = 0
kernel.sysrq = 0
kernel.core_uses_pid = 1
net.ipv4.tcp_syncookies = 1
kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.shmmax = 68719476736
kernel.shmall = 4294967296
net.ipv4.tcp_max_tw_buckets = 50000
net.ipv4.tcp_sack = 1
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_rmem = 4096  87380   15564800
net.ipv4.tcp_wmem = 4096  87380   4194304
net.ipv4.tcp_keepalive_intvl = 65
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_fin_timeout = 1
net.ipv4.tcp_keepalive_time = 1200
net.ipv4.tcp_mem = 94500000 915000000 927000000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_syn_retries = 1
net.ipv4.tcp_tw_recycle = 1
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.netdev_max_backlog = 327680
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 2000000
net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.ipv4.tcp_retries1 = 2
net.ipv4.tcp_retries2 = 5
net.ipv4.tcp_orphan_retries = 2
net.ipv4.tcp_max_orphans = 327680
net.ipv4.ip_local_port_range = 5000 65000
vm.dirty_background_bytes = 52428800
vm.dirty_bytes = 52428800
vm.dirty_ratio = 0
vm.dirty_background_ratio = 0


net.netfilter.nf_conntrack_max=1048576
net.nf_conntrack_max=1048576
net.netfilter.nf_conntrack_tcp_timeout_fin_wait=30
net.netfilter.nf_conntrack_tcp_timeout_time_wait=30
net.netfilter.nf_conntrack_tcp_timeout_close_wait=15
net.netfilter.nf_conntrack_tcp_timeout_established=300
```


### 场景二 1分钟并发量超过5W+


```
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-arptables = 1
kernel.msgmnb=65536
# Controls the maximum size of a message, in bytes
kernel.msgmax=65536
#
# # Controls the maximum shared segment size, in bytes
kernel.shmmax=68719476736
#
# # Controls the maximum number of shared memory segments, in pages
kernel.shmall=4294967296
vm.swappiness=0
net.ipv4.neigh.default.gc_stale_time=120
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
net.ipv4.conf.default.arp_announce=2
net.ipv4.conf.all.arp_announce=2
net.ipv4.tcp_max_tw_buckets=5000
net.ipv4.tcp_syncookies=1
net.ipv4.tcp_max_syn_backlog=1024
net.ipv4.tcp_synack_retries=2
net.ipv4.conf.lo.arp_announce=2
#
# #表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭；
net.ipv4.tcp_tw_reuse=1
# ##表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；
net.ipv4.tcp_tw_recycle=0
# ##表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭；最低30s
net.ipv4.tcp_fin_timeout=30
# ##修改系統默认的 TIMEOUT 时间。
net.ipv4.ip_local_port_range=10000 65000
net.ipv4.tcp_keepalive_time=1200
net.ipv4.tcp_keepalive_probes=3
net.ipv4.tcp_keepalive_intvl=15
net.ipv4.tcp_timestamps=1
#
net.nf_conntrack_max=5242888
```
