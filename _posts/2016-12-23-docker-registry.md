---
layout: post
categories: Docker
title: docker registry 
date: 2016-12-23 21:16:52 +0800
description: docker
keywords: docker registry
---

关于私有仓库配置docker官网给出的参考示例 其实已经很详尽了，但是还是不免在配置过程中 踩坑。这里根据官网文档整理了配置示例供大家参考！

- host文件 

```sh

192.168.11.10   hub.test.io 
192.168.11.20   node1
192.168.11.30   node2

```

### 时间同步(分别执行)：

``` sh
[root@hub ]# yum install -y ntpdate && ntpdate cn.pool.ntp.org
[root@node1 ]# yum install -y ntpdate && ntpdate cn.pool.ntp.org
[root@hub ]# crontab -l
*/5 * * * * ntpdate cn.pool.ntp.org
```

## 1.仓库服务端设置 hub.test.io：
``` sh
[root@hub ]# docker pull registry
[root@hub ]# cat /etc/hosts   //内网没有DNS情况下修改hosts,修改主机名
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.11.10 hub.test.io
[root@hub ~]# hostname    
hub.test.io
说明：如果私有仓库用的不是域名而是IP，请加上此设置：
- # sed -i '/^\[ v3_ca \]$/a subjectAltName = IP:192.168.10.10' /etc/ssl/openssl.cnf
[root@hub ]# mkdir /opt/registry && cd /opt/registry
[root@hub registry]# mkdir auth certs
[root@hub registry]# docker run --entrypoint htpasswd registry:latest -Bbn username password > auth/htpasswd  //自行更换用户名密码
[root@hub registry]# cat auth/htpasswd   // 下面一行为空，切记不要修改生成后的文件
admin:$2y$05$LSRMXpIbnvnj8ErzbRvKq.F04Qf3oajP7dFWQIjJBrFAoDXKM1I16

[root@hub registry]# openssl req -newkey rsa:4096 -nodes -sha256 -keyout certs/ca.key -x509 -days 365 -out certs/ca.crt
Generating a 4096 bit RSA private key
........
........
....++writing new private key to certs/ca.key
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:
State or Province Name (full name) []:
Locality Name (eg, city) [Default City]:
Organization Name (eg, company) [Default Company Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your servers hostname) []:hub.test.io
Email Address []:
[root@hub registry]# mkdir -p /etc/docker/certs.d/hub.test.io && cp certs/ca.crt /etc/docker/certs.d/hub.test.io
[root@hub registry]# ls /etc/docker/certs.d/hub.test.io/
ca.crt
[root@hub registry]# systemctl daemon-reload && systemctl restart docker
[root@hub registry]# cat start.sh
#!/bin/bash
docker run -d \
-p 443:5000 \
--name registry \
--restart=always \
-v /var/lib/registry:/var/lib/registry \
-v `pwd`/auth:/auth \
-e REGISTRY_AUTH=htpasswd \
-e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
-e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
-v `pwd`/certs:/certs \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/ca.crt \
-e REGISTRY_HTTP_TLS_KEY=/certs/ca.key \
registry:latest
[root@hub registry]# tree   //最终的目录结构
.
├── auth
│   └── htpasswd
├── certs
│   ├── ca.crt
│   └── ca.key
└── start.sh
[root@hub registry]# docker login hub.test.io
Username: admin
Password: 
Login Succeeded
[root@hub registry]# docker tag pause-amd64:3.0 hub.test.io/pause-amd64:3.0
[root@hub registry]# docker push hub.test.io/pause-amd64:3.0
The push refers to a repository [hub.test.io/pause-amd64]
5f70bf18a086: Pushed 
41ff149e94f2: Pushed 
3.0: digest: sha256:ec6581792f828ab138bc7ed65205dbd4d7df966249179b7afbb9f6cac729771b size: 939
```
## 客户端：
[Test an insecure registry](https://docs.docker.com/registry/insecure/)

``` sh
1.仓库配置(私有证书):
[root@node1 ]# mkdir /etc/docker/certs.d/hub.test.io 
[root@node1 ]# scp hub.test.io:/etc/docker/certs.d/hub.test.io/ca.crt /etc/docker/certs.d/hub.test.io

### 或者通过添加仓库地址配置: ###
[root@node1 ]# vim /lib/systemd/system/docker.service
.....
--insecure-registry="hub.test.io"
.....

2.登陆仓库并下载镜像：
[root@node1 ]# docker login hub.test.io
[root@node1 ]# docker pull hub.test.io/pause-amd64:3.0
3.0: Pulling from pause-amd64
a3ed95caeb02: Pull complete 
f11233434377: Pull complete 
Digest: sha256:ec6581792f828ab138bc7ed65205dbd4d7df966249179b7afbb9f6cac729771b
Status: Downloaded newer image for hub.test.io/pause-amd64:3.0
至此私有仓库就搭建完成了
```

更多配置详解 请查看 [Docker官方registry配置文档](https://docs.docker.com/registry/)

