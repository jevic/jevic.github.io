---
layout: post
categories: Docker
title: Rancher 部署Traefik 实现微服务快速发现 
date: 2018-02-28 21:12:36 +0800
description: traefik-rancher
keywords: rancher,docker,traefik
---

>Traefik是一个为了让部署微服务更加便捷而诞生的现代HTTP反向代理、负载均衡工具。本文将向你展示如何在Rancher上简单快速地部署Traefik，实现微服务的快速发现

![](https://docs.traefik.io/img/traefik.logo.png)

## Traefik 是什么?
Traefik 是一个为了让部署微服务更加便捷而诞生的现代HTTP反向代理、负载均衡工具。它支持多种后台 (Rancher、Docker、Swarm、Kubernetes、Marathon、Mesos、Consul、Etcd、Zookeeper、BoltDB、Rest API、file…) 来自动、动态的刷新配置文件，以实现快速地服务发现
![](https://docs.traefik.io/img/architecture.png)
## 特性
- 它非常快
- 无需安装其他依赖，通过Go语言编写的单一可执行文件
- 支持 Rest API
- 多种后台支持：Rancher、Docker、Swarm、Kubernetes、Marathon、Mesos、Consul、Etcd，并且还会更多
- 后台监控，可以监听后台变化进而自动化应用新的配置文件设置
- 配置文件热更新。无需重启进程
- 正常结束http连接
- 后端断路器
- 轮询，rebalancer 负载均衡
- Rest Metrics
- 支持最小化 官方 docker 镜像
- 后台支持SSL
- 前台支持SSL（包括SNI）
- 清爽的AngularJS前端页面
- 支持Websocket
- 支持HTTP/2
- 网络错误重试
- 支持Let’s Encrypt (自动更新HTTPS证书)
- 高可用集群模式

## 清爽的界面
Traefik 拥有一个基于AngularJS编写的简单网站界面。
![](http://traefik.cn/frontend/images/web.frontend.png)
![](http://traefik.cn/frontend/images/traefik-health.png)

## Rancher-Traefik 部署
### 添加节点主机标签
![](http://ok6h8mla5.bkt.clouddn.com/traefik-demo-01-02.png)
### 安装 Traefik 应用
- 点击查看详情进入配置界面, http port 端口改为80,其他配置保持默认
- 最后点击启动；
![](http://ok6h8mla5.bkt.clouddn.com/traefik-demo-02.png)
![](http://ok6h8mla5.bkt.clouddn.com/traefik-demo-03.png)
#### 修改rancher 默认traefik.domain 
>由于rancher提供的镜像默认traefik.domain 为: rancher.internal,需要修改为自己所使用的方可
>如果服务指定了 traefik.frontend.rule = Host:xxxx.com 标签,则会使用自定义前端域名访问
>建议直接指定访问域名

- 方式一：
    - 升级服务 加入环境变量: TRAEFIK_RANCHER_DOMAIN = jevic.cn
![](http://ok6h8mla5.bkt.clouddn.com/traefik-demo-20.png)

- 方式二:
    - 直接基于社区Dockerfile 修改域名
    - https://github.com/rawmind0/alpine-traefik.git
    - 路径: alpine-traefik/root/opt/traefik/bin/traefik.toml.sh
![](http://ok6h8mla5.bkt.clouddn.com/traefik-demo-21.png)
![](http://ok6h8mla5.bkt.clouddn.com/traefik-demo-22.png)

- 进入 应用|用户 视图,查看应用；
![](http://ok6h8mla5.bkt.clouddn.com/traefik-demo-04.png)
- 此时可以访问 Traefik_host:8000 管理后台

### 运行 demo 示例
- 新建一个名为 demo 的空应用栈；
- 在 demo 中添加一个名为 nginx 的服务
- Traefik默认强制开启健康检查，所有只有健康的服务才会被注册到Traefik上。在健康检查中配置健康检查
![](http://ok6h8mla5.bkt.clouddn.com/traefik-demo-05.png)

#### 标签设定
##### 默认标签设置
- traefik.enable = true 可以理解为是否把此服务注册到traefik的一个开关;
    - true: the service will be published as service_name.stack_name.traefik_domain
    - stack: the service will be published as stack_name.traefik_domain. WARNING: You could have collisions inside services within your stack
    - false: the service will not be published
- traefik.domain = jevic.cn 一个适用于所有服务访问的主域名，可以设置多个用逗号隔开;
- traefik.alias = nginx 服务别名，可以理解为主域名下的二级域名，可以设置多个用逗号隔开；
- traefik.port = 80 告诉traefik 服务暴露的端口号；

![](http://ok6h8mla5.bkt.clouddn.com/traefik-demo-06.png)

- 此时 Traefik管理后台 显示的前端访问域名为:

``` sh
http://${service_name}.${stack_name}.${traefik.domain}:${http_port}
https://${service_name}.${stack_name}.${traefik.domain}:${https_port}
```
![](http://ok6h8mla5.bkt.clouddn.com/traefik-demo-07.png)

##### 自定义前端Host [建议使用此方式]
- traefik.frontend.rule = Host:ops.jevic.cn 直接定义前端访问域名,多个使用','分割
- traefik.port = 80
- traefik.enable = true

![](http://ok6h8mla5.bkt.clouddn.com/traefik-demo-06-2.png)

- 前端访问域名:
![](http://ok6h8mla5.bkt.clouddn.com/traefik-demo-07-02.png)

#### 最终,应用栈如下所示
![](http://ok6h8mla5.bkt.clouddn.com/traefik-demo-11.png)

### 访问测试
- 如上面图示，控制台显示了当前前端访问host 域名以及后端的节点状态
- 测试域名访问:
    - app.jevic.cn
    - ops.jevic.cn
    - nginx.demo.jevic.cn
![](http://ok6h8mla5.bkt.clouddn.com/traefik-demo-13.png)
![](http://ok6h8mla5.bkt.clouddn.com/traefik-demo-12.png)
![](http://ok6h8mla5.bkt.clouddn.com/traefik-demo-09.png)

- http://192.168.3.27:8000/dashboard/#/health
![](http://ok6h8mla5.bkt.clouddn.com/traefik-demo-10.png)


### 参考链接
- [alpine-traefik](https://github.com/rawmind0/alpine-traefik/tree/master/rancher)
- [Traefik.io](https://docs.traefik.io/)
- [traefik.cn](https://docs.traefik.cn/)


转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
