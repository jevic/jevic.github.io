---
layout: post
categories: Docker
title: 自动构建docker registry仓库 shell
date: 2017-01-23 19:10:35 +0800
description: docker registry
keywords: docker registry
---


## 运行 registry

``` bash
[root@master ~]# cat build.sh 
#!/bin/bash
## Docker registry 构建    
## Version : 2.0                      
## author: jevic

set -e
## 镜像名称
REGS="registry:latest"
DATES=`date '+%Y-%m-%d %H:%M:%S'`

## 判断是否已存在仓库运行
PORTS=`netstat -tnlp|grep 443|wc -l`
PSS=`docker ps |grep 'registry'|wc -l`

if [ $PORTS -eq 1 -a $PSS == 0 ];then
   echo -e "\033[31m <-- 443端口已经被其他程序占用 -->\033[0m" 
   exit "`echo $RANDOM|cut -c 1,2`"
elif [ $PORTS -eq 1 -a $PSS == 1 ] 
then
   echo -e "\033[31m <-- 已有仓库在运行 --> \033[0m"
   exit "`echo $RANDOM|cut -c 1,2`"
else
  echo -e "\033[35m--- Docker Registry ---\033[0m"
fi

## 运行容器
Regs () {
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
$REGS
}

## 设定变量
echo -e "\033[32m请输入用户名:\033[0m" && read -p '' USERS
echo -e "\033[32m请输入密码:\033[0m" && read -p '' PASSWD
echo -e "\033[32m请输入域名,请勿输入IP:\033[0m" && read -p '' DOMAIN
echo -e "\033[32m请输入Email:\033[0m" && read -p '' EMAIL
echo ''
echo -e "创建时间: ${DATES}\nuser: $USERS\npasswd: $PASSWD\n域名: $DOMAIN" |tee registry.log
## 初始化目录
## 
if [ ! -f `pwd`/auth/htpasswd -o ! -d `pwd`/certs/ca.crt ];then
    mkdir -p `pwd`/{auth,certs}
fi

## 生成密码文件
docker run --entrypoint htpasswd ${REGS} -Bbn ${USERS} ${PASSWD} > auth/htpasswd

## 配置证书文件
openssl req -newkey rsa:4096 -nodes -sha256 -keyout certs/ca.key -x509 -days 365 -out certs/ca.crt <<EOF
CN
GD
SZ
SOFT
APP
$DOMAIN
$EMAIL
EOF

if [ $? == 0 ];then
   echo -e "\n\033[32m<-- 生成证书成功 -->\033[0m"
else
   echo -e "\n\033[31m<-- 生成证书失败 -->\033[0m"
   exit 11
fi

if [ -f /etc/docker/certs.d/${DOMAIN}/ca.crt ];then
     rm -rf /etc/docker/certs.d/${DOMAIN}
else 
   mkdir -p /etc/docker/certs.d/${DOMAIN}
   cp -a certs/ca.crt /etc/docker/certs.d/${DOMAIN}/
fi

## 重启 Docker
echo -e "\033[32m*** 重启docker ***\n....\033[0m"
systemctl daemon-reload && systemctl restart docker

## Run
echo -e "\033[32m*** 运行仓库 ***\033[0m"
Regs
if [ $? -eq 0 ];then
  echo -e "\033[32m*** 仓库构建完成 ***\033[0m"
else
  echo -e "\033[31m*** 仓库构建失败 ***\033[0m"
  exit 1
fi

```

## 自动登录脚本
* 需要安装expect
* yum install -y expect

``` bash
[root@master ~]# cat login
#!/bin/expect
set USER "admin"
set PASS "admin"

spawn docker login jevic.hub
expect "Username*" 
send "$USER\r"
expect "Password:"
send "$PASS\r"
interact
[root@master ~]# ./login 
spawn docker login jevic.hub
Username: admiin
Password: 
Login Succeeded


```

### API:
[Registry API](https://docs.docker.com/registry/spec/api/)

```
curl -u admin:admin --cacert ca.crt https://hub.jevic.cn/v2/_catalog

curl -u admin:admin --cacert ca.crt https://hub.jevic.cn/v2/nginx/tags/list

curl -u admin:admin https://hub.jevic.cn/v2/nginx/manifests/v1

```

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
