---
layout: post
categories: Kubernetes
title: kubernetes 高可用集群api-server请求异常
date: 2019-03-08 20:10:46 +0800
description: kubernetes
keywords: kubernetes
---

高可用kubernetes 集群下关于使用nginx代理服务连接超时错误处理


- 报错信息如下图:
![](https://raw.githubusercontent.com/jevic/images/master/kubernetes/k8s-nginx-proxy-error.png)


- [issues](https://github.com/rancher/rancher/issues/14971)

 
I've checked rancher nginx-proxy config and it indeed reports proxy_timeout 5;. When I change the config in-place (setting proxy_timeout to 500) and reload nginx, it works OK.



- 通过上面的问题描述可以得出: 

测试发现直连api-server 接口时服务正常。

由于nginx-proxy `proxy_timeout` 默认超时时间太短,导致了连接api-server 超时请求失败;

所以需要将此 `proxy_timeout` 参数适当增大,服务恢复正常;
