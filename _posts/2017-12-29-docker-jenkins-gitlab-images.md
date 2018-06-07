---
layout: post
categories: Docker
title: jenkens-gitlab-docker 自动化构建镜像(一)
date: 2017-12-26 21:51:44 +0800
description: jenkins-docker-gitlab
keywords: docker jenkins gitlab
---


>- gitlab来管理dockerfile 源码
- 使用jenkins来构建部署应用的docker 镜像并自动push到私有仓库
    - 通过执行相关自定义操作来进行应用的自动测试、部署应用操作
- 进而减少不必要的人为手动操作,驱动自动化的应用部署极大提升效率,实现持续集成与构建!


- 此示例演示了镜像的自动构建操作，更多的例如自动部署应用 后续补充......



## 安装 Docker
- 省略...... 请自行查看docker官方文档

## 运行Jenkins
- https://hub.docker.com/r/jevic/jenkins/

``` sh
docker run -d --name jenkins \
-p 8080:8080 -p 50000:50000 \
-v /data/ci/jenkins:/var/jenkins_home \
-v /var/run/docker.sock:/var/run/docker.sock \
jevic/jenkins:docker-latest
```


## 运行gitlab-ce 中文社区版 (9.x 版本)
- https://hub.docker.com/r/twang2218/gitlab-ce-zh/
 
```
version: '2'
services:
    gitlab:
      image: 'twang2218/gitlab-ce-zh:9.2.10'
      restart: unless-stopped
      hostname: 'gitlab.jevic.cn'
      environment:
        TZ: 'Asia/Shanghai'
      ports:
        - '80:80'
        - '443:443'
        - '22:22'
      volumes:
        - config:/etc/gitlab
        - data:/var/opt/gitlab
        - logs:/var/log/gitlab
volumes:
    config:
    data:
    logs:
```

## 配置jenkins

### 安装插件
![](http://ok6h8mla5.bkt.clouddn.com/jenkins-docker-plugin.png)
![](http://ok6h8mla5.bkt.clouddn.com/jenkins-gitlab-plugs.png)

### 系统设置

#### 配置私有仓库地址以及认证信息
![](http://ok6h8mla5.bkt.clouddn.com/jenkins-registry-user.png)

#### 设置docker 服务
- 新增一个云  选择 docker
- unix:///var/run/docker.sock
![](http://ok6h8mla5.bkt.clouddn.com/jenkins-docker-cloud.png)

#### 获取用户Token

返回主面板 -> 用户 -> 点击admin -> 设置

![](http://ok6h8mla5.bkt.clouddn.com/jenkins-admon-token.png)

#### 创建任务
![](http://ok6h8mla5.bkt.clouddn.com/jenkins-jobs-01.png)
##### 源码管理
![](http://ok6h8mla5.bkt.clouddn.com/jenkins-gitlab-nginx-master.png)

##### 触发器
- 触发远程构建 钩子

Use the following URL to trigger build remotely:
JENKINS_URL/job/webhook/build?token=TOKEN_NAME 或者 
/buildWithParameters?token=TOKEN_NAME


![](http://ok6h8mla5.bkt.clouddn.com/jenkins-chufaqi.png)

##### 构建环境默认
##### 构建
- 以构建次数为标签 hub.jevic.com/nginx/nginx:${BUILD_NUMBER}
![](http://ok6h8mla5.bkt.clouddn.com/jenkins-docker-build.png)

## gitlab 项目配置

### 创建新的项目 nginx
- 例如: http://jevic@gitlab.jevic.cn/jevic/nginx.git

### 设置 -> 集成webhook
- URL 格式如下: 
    - http://ci.jevic.cn/buildByToken/build?job=webhook&token=[API-TOKEN]
- 示例:
    - http://ci.jevic.cn:8080/buildByToken/build?job=webhook&token=494e32323dfsdfaeb0c49f8sdfa232615   

![](http://ok6h8mla5.bkt.clouddn.com/jenkins-gitlab-webhook.png)

- 测试
![](http://ok6h8mla5.bkt.clouddn.com/jenkins-gitlab-webhook-02.png)

## 运行任务
1. push 代码到仓库
2. 查看jenkins 构建任务运行状态

```
工作目录
[root@jevic.cn webhook]# pwd
/data/ci/jenkins/workspace/webhook
[root@jevic.cn webhook]# ls
Dockerfile  README.md

```
![](http://ok6h8mla5.bkt.clouddn.com/jenkens-webhook-jobs-ok%20%281%29.png)
3. 查看仓库
![](http://ok6h8mla5.bkt.clouddn.com/jenkins-registry-harbor.png)


转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
