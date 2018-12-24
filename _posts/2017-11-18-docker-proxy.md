---
layout: post
categories: Docker,Kubernetes
title: Docker 配置代理访问
date: 2017-11-18 19:56:06 +0800
description: Docker
keywords: Docker
---

[Configure Docker to use a proxy server](https://docs.docker.com/network/proxy/#use-environment-variables)

## proxy
代理服务使用shadowsocks+Privoxy

``` bash
# vim /lib/systemd/system/docker.service
Environment="HTTP_PROXY=http://127.0.0.1:8118" "NO_PROXY=localhost,127.0.0.1,.jevic.cn,daocloud.io"
```

## pull
 
``` bash
# systemctl daemon-reload
# systemctl restart docker
# docker pull k8s.gcr.io/cpvpa-amd64:v0.6.0
v0.6.0: Pulling from cpvpa-amd64
280aca6ddce2: Pull complete
bd34b7466228: Pull complete
Digest: sha256:cfe7b0a11c9c8e18c87b1eb34fef9a7cbb8480a8da11fc2657f78dbf4739f869
Status: Downloaded newer image for k8s.gcr.io/cpvpa-amd64:v0.6.0
```
