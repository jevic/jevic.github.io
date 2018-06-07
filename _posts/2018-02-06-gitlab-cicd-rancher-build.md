---
layout: post
categories: OPS
title: gitlab CI/CD 自动化构建Rancher容器服务
date: 2018-02-06 19:07:22 +0800
description: gitlab CI/CD 自动化构建Rancher容器服务
keywords: gitlab cicd 自动化
---

# 概述
- gitlab
- rancher

## Rancher
[Rancher](https://www.cnrancher.com/)是一个开源的企业级容器管理平台。通过Rancher，企业再也不必自己使用一系列的开源软件去从头搭建容器服务平台。Rancher提供了在生产环境中使用的管理Docker和Kubernetes的全栈化容器部署与管理平台。
![](http://rancher.com/docs/img/rancher/rancher_overview_2.png)
## Gitlab CI/CD
![](https://docs.gitlab.com/ee/ci/img/cicd_pipeline_infograph.png)
### 1. Gitlab
[GitLab](https://docs.gitlab.com/ee/README.html) 是一个利用Ruby on Rails开发的开源应用程序，实现一个自托管的Git项目仓库，可通过Web界面进行访问公开的或者私人项目。
它拥有与GitHub类似的功能，能够浏览源代码，管理缺陷和注释。可以管理团队对仓库的访问，它非常易于浏览提交过的版本并提供一个文件历史库。团队成员可以利用内置的简单聊天程序（Wall）进行交流。它还提供一个代码片段收集功能可以轻松实现代码复用，便于日后有需要的时候进行查找。

### 2. Gitlab-CI
Gitlab-CI是GitLab Continuous Integration（Gitlab持续集成）的简称。
从Gitlab的8.0版本开始，gitlab就全面集成了Gitlab-CI,并且对所有项目默认开启。
只要在项目仓库的根目录添加.gitlab-ci.yml文件，并且配置了Runner（运行器），那么每一次合并请求（MR）或者push都会触发CI pipeline。

### 3. Gitlab-runner
Gitlab-runner 是.gitlab-ci.yml脚本的运行器，Gitlab-runner是基于Gitlab-CI的API进行构建的相互隔离的机器（或虚拟机）。GitLab Runner 不需要和Gitlab安装在同一台机器上，但是考虑到GitLab Runner的资源消耗问题和安全问题，也不建议这两者安装在同一台机器上。

Gitlab Runner分为两种，Shared runners和Specific runners。
Specific runners只能被指定的项目使用，Shared runners则可以运行所有开启 Allow shared runners选项的项目。

### 4. Pipelines
Pipelines是定义于.gitlab-ci.yml中的不同阶段的不同任务。
我把Pipelines理解为流水线，流水线包含有多个阶段（stages），每个阶段包含有一个或多个工序（jobs），比如先购料、组装、测试、包装再上线销售，每一次push或者MR都要经过流水线之后才可以合格出厂。而.gitlab-ci.yml正是定义了这条流水线有哪些阶段，每个阶段要做什么事。

### 5. Badges
徽章，当Pipelines执行完成，会生成徽章，你可以将这些徽章加入到你的README.md文件或者你的网站。

## 环境配置
### Rancher
> 具体安装请查看[官网文档](http://rancher.com/docs/rancher/v1.6/zh/)

#### 运行 Nginx 服务
##### 1. 添加私有仓库并添加证书
![](http://ok6h8mla5.bkt.clouddn.com/rancher-gitlab-cicd-registry.jpg)
![](http://ok6h8mla5.bkt.clouddn.com/rancher-gitlab-cicd-registry-02.jpg)
##### 2. 创建应用服务
- 网络模式: `主机`，如果设置为manage 需要配置`负载均衡`
- 切记要对服务进行标签设定
![](http://ok6h8mla5.bkt.clouddn.com/rancher-gitlab-cicd-nginx-01.jpg)

- 访问
![](http://ok6h8mla5.bkt.clouddn.com/rancher-gitlab-cicd-web-01.jpg)
##### 3. 添加接收器 webhooks
![](http://ok6h8mla5.bkt.clouddn.com/rancher-gitlab-cicd-webhook.jpg)
![](http://ok6h8mla5.bkt.clouddn.com/rancher-gitlab-cicd-webhook-02.jpg)

### Gitlab 
>中文社区版 [docker hub](https://hub.docker.com/r/twang2218/gitlab-ce-zh/)
>安装步骤省略.....

### gitlab-runner
>注: 主版本要与gitlab 一致否则无法添加!!!

>[高级配置](https://gitlab.com/gitlab-org/gitlab-runner/blob/master/docs/configuration/advanced-configuration.md)


- 官方镜像并未配置docker CLI,如有需要安装即可，否则直接使用官方镜像run 即可。
- 此处基于官方镜像配置docker CLI，并运行

##### 安装docker 命令

``` sh 
# cat Dockerfile
FROM gitlab/gitlab-runner:v9.2.2
RUN curl -O https://get.docker.com/builds/Linux/x86_64/docker-latest.tgz  \
    && tar zxvf docker-latest.tgz \
    && cp docker/docker /usr/local/bin/ \
    && rm -rf docker docker-latest.tgz  
# docker build -t gitlab-runner:docker .
```

##### 运行

``` sh
docker run -d --name gitlab-runner --restart always \
-v /srv/gitlab-runner/config:/etc/gitlab-runner \
-v /var/run/docker.sock:/var/run/docker.sock \
gitlab-runner:docker

```

##### 配置sudo 
>由于需要执行docker 命令所以需要配置sudo 免密码执行权限

``` sh
# docker exec -it 0c115c075b32 /bin/bash
root@0c115c075b32:/# cat /etc/sudoers |grep gitlab
gitlab-runner    ALL=(ALL)       NOPASSWD: ALL

```

##### 注册 Runners
> 根据提示输入指定参数即可

> URL  gitlab URL

> token 注册认证token 

> description 描述信息，自定义

> tags  用来区分多个runners,自定义

> executor 根据需要选择即可，此处选择的shell

![](http://ok6h8mla5.bkt.clouddn.com/rancher-gitlab-cicd-runners.jpg)

- register

``` sh
root@0c115c075b32:/# gitlab-ci-multi-runner register
Running in system-mode.

Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
http://test.gitlab.com/ci
Please enter the gitlab-ci token for this runner:
xxxxxxxx-xxxxxxx
Please enter the gitlab-ci description for this runner:
[0c115c075b32]: t197
Please enter the gitlab-ci tags for this runner (comma separated):
t197
Whether to run untagged builds [true/false]:
[false]: 直接回车
Whether to lock Runner to current project [true/false]:
[false]: 直接回车
INFO[0034] fcf5c619 Registering runner... succeeded
Please enter the executor: shell, docker, docker-ssh, ssh?
shell
INFO[0037] Runner registered successfully. Feel free to start it, but if it's
running already the config should be automatically reloaded!
```

- 此时打开gitlab CI/CD 设置页面 即可查看到注册的Runners
![](http://ok6h8mla5.bkt.clouddn.com/rancher-gitlab-cicd-set-01.jpg)

- 设置构建时的环境变量
![](http://ok6h8mla5.bkt.clouddn.com/rancher-gitlab-cicd-set-02.jpg)

##### 至此所有准备工作完毕，下面开始首次构建
### CI/CD Test
#### dockerfile Version0.1
![](http://ok6h8mla5.bkt.clouddn.com/rancher-gitlab-cicd-web-01.jpg)
>下面将通过gitlab ci/cd 结合rancher 升级服务到version 0.2

#### update Version0.2
>编辑更新Dockerfile

``` sh
FROM nginx:alpine

RUN echo "<h2>Gitlab CI/CD on Rancher NginxApp Version 0.2....<h2>" > /usr/share/nginx/html/index.html

```
#### Rancher 服务升级触发脚本

``` sh
#!/bin/bash
## rancher 触发器
Webhook=$1
## 仓库名称
repo=$2

curl -XPOST -H 'content-type: application/json' \
"$Webhook" \
-d '{
    "push_data": {
        "tag": "master"
    },
    "repository": {
        "repo_name": "'$repo'"
    }
}'

```

#### .gitlab-ci.yml
- [官方配置文件详解](https://docs.gitlab.com/ce/ci/yaml/README.html)
- [中文说明](http://blog.csdn.net/wmq880204/article/details/70141771)
- 引用Runners 添加的环境变量
- 尽量使用环境变量来引用各个参数

``` sh
jay➜/tmp/cicd(master✗)» cat .gitlab-ci.yml                                                                                                                             [16:19:16]
variables:
  Tags: master

stages:
  - build
  - release
# 环境
before_script:
  - sudo docker --version
  - sudo uname -a
# 构建镜像
build:
  stage: build
  script:
    - sudo echo "$repo:$Tags"
    - sudo docker build --pull -t "$repo:$Tags" .
    - sudo docker login -u "$USER" -p "$PASSWD" $registry
    - sudo docker push "$repo:$Tags"
    - sudo docker rmi -f "$repo:$Tags"
# 升级rancher服务
release:
  stage: release
  script:
    - sh update.sh $Webhook $repo
```

#### 提交commit

``` sh
jay➜/tmp/cicd(master✗)» git add --all                                                                                                                                  [16:28:37]

jay➜/tmp/cicd(master✗)» git commit -m "version 0.2"                                                                                                                    [16:29:25]
[master 6647463] version 0.2
 2 files changed, 4 insertions(+), 3 deletions(-)

jay➜/tmp/cicd(master)» git push origin master                                                                                                                          [16:30:20]
Counting objects: 4, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (4/4), done.
Writing objects: 100% (4/4), 412 bytes | 0 bytes/s, done.
Total 4 (delta 3), reused 0 (delta 0)
To http://test.gitlab.com/jevic/cicd.git
   3dd8d23..6647463  master -> master
```

#### 查看项目状态
![](http://ok6h8mla5.bkt.clouddn.com/rancher-gitlab-cicd-ok.jpg)
![](http://ok6h8mla5.bkt.clouddn.com/rancher-gitlab-cicd-02.jpg)
![](http://ok6h8mla5.bkt.clouddn.com/rancher-gitlab-cicd-03.jpg)

#### 页面访问
![](http://ok6h8mla5.bkt.clouddn.com/rancher-gitlab-cicd-web-02.jpg)


转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
