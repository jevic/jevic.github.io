---
layout: post
categories: Docker Kubernetes
title: kubernetes 核心概念简述
date: 2018-08-01 16:42:16 +0800
description: Docker,kubernetes
keywords: docker,kubernetes
---

>Kubernetes主要由以下几个核心组件组成：

- etcd保存了整个集群的状态；
- apiserver提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制；
- controller manager负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
- scheduler负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上；
- kubelet负责维护容器的生命周期，同时也负责Volume（CVI）和网络（CNI）的管理；
- Container runtime负责镜像管理以及Pod和容器的真正运行（CRI）；
- kube-proxy负责为Service提供cluster内部的服务发现和负载均衡；
 
![](https://raw.githubusercontent.com/jevic/images/master/kubernetes/architecture.png)
### Master
Master节点上面主要由四个模块组成：`APIServer`、`scheduler`、`controller manager`、`etcd`。

`APIServer`。APIServer负责对外提供RESTful的Kubernetes API服务，它是系统管理指令的统一入口，任何对资源进行增删改查的操作都要交给APIServer处理后再提交给etcd。如架构图中所示，kubectl（Kubernetes提供的客户端工具，该工具内部就是对Kubernetes API的调用）是直接和APIServer交互的。

`schedule`。scheduler的职责很明确，就是负责调度pod到合适的Node上。如果把scheduler看成一个黑匣子，那么它的输入是pod和由多个Node组成的列表，输出是Pod和一个Node的绑定，即将这个pod部署到这个Node上。Kubernetes目前提供了调度算法，但是同样也保留了接口，用户可以根据自己的需求定义自己的调度算法。

`controller manager`。如果说APIServer做的是“前台”的工作的话，那controller manager就是负责“后台”的。每个资源一般都对应有一个控制器，而controller manager就是负责管理这些控制器的。比如我们通过APIServer创建一个pod，当这个pod创建成功后，APIServer的任务就算完成了。而后面保证Pod的状态始终和我们预期的一样的重任就由controller manager去保证了。

`etcd`。etcd是一个高可用的键值存储系统，Kubernetes使用它来存储各个资源的状态，从而实现了Restful的API。

### Node
每个Node节点主要由三个模块组成：`kubelet`、`kube-proxy`、`runtime`。

`runtime`。runtime指的是容器运行环境，目前Kubernetes支持docker和rkt两种容器。

`kube-proxy`。该模块实现了Kubernetes中的服务发现和反向代理功能。反向代理方面：kube-proxy支持TCP和UDP连接转发，默认基于Round Robin算法将客户端流量转发到与service对应的一组后端pod。服务发现方面，kube-proxy使用etcd的watch机制，监控集群中service和endpoint对象数据的动态变化，并且维护一个service到endpoint的映射关系，从而保证了后端pod的IP变化不会对访问者造成影响。另外kube-proxy还支持session affinity。

`kubelet`。Kubelet是Master在每个Node节点上面的agent，是Node节点上面最重要的模块，它负责维护和管理该Node上面的所有容器，但是如果容器不是通过Kubernetes创建的，它并不会管理。本质上，它负责使Pod得运行状态与期望的状态一致。

至此，Kubernetes的Master和Node就简单介绍完了。下面我们来看Kubernetes中的各种资源/对象。


## label 

添加

```
kubectl label nodes master219 myapp=flask-app
```

删除

```
kubectl label nodes master219 myapp-
```

>许多资源支持内嵌字段定义其使用的标签选择器:

- matchLabels:直接给定键值 
- matchExpressions:基于给定的表达式来定义使用标签选择器
    - {key:"KEY", operator:"OPERATOR",

- values:[VAL1,VAL2,...]} 操作符:
    - In, NotIn:  values字段的值必须为非空列表; 
    - Exists, NotExists:  values字段的值必须为空列表

## Pod
### Pod 配置项
镜像:
Always：每次都下载最新的镜像
Never：只使用本地镜像，从不下载
IfNotPresent：只有当本地没有的时候才下载镜像


Host网络: 
一些特殊场景下，容器必须要以host方式进行网络设置（如接收物理机网络才能够接收到的组播流），在Pod中也支持host网络的设置，如：kubectl explain Pod.spec.hostNetwork；


affinity: 调度器; kubectl explain Pod.spec.affinity

## Controllers
关于pod 目前所支持的控制器请查看[官网文档](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)


### ReplicationController:

Replication Controller（RC）是Kubernetes中的另一个核心概念，应用托管在Kubernetes之后，Kubernetes需要保证应用能够持续运行，这是RC的工作内容，它会确保任何时间Kubernetes中都有指定数量的Pod在运行。在此基础上，RC还提供了一些更高级的特性，比如滚动升级、升级回滚等。

### ReplicaSet: 

这里所说的replica set，可以被认为 是“升级版”的Replication Controller。也就是说。replica set也是用于保证与label selector匹配的pod数量维持在期望状态。区别在于，replica set引入了对基于子集的selector查询条件，而Replication Controller仅支持基于值相等的selecto条件查询。这是目前从用户角度肴，两者唯一的显著差异。 社区引入这一API的初衷是用于取代vl中的Replication Controller，也就是说．当v1版本被废弃时，Replication Controller就完成了它的历史使命，而由replica set来接管其工作。虽然replica set可以被单独使用，但是目前它多被Deployment用于进行pod的创建、更新与删除。　

### DaemonSet

DaemonSet能够让所有（或者特定）的节点运行同一个pod。

当节点加入到K8S集群中，pod会被（DaemonSet）调度到该节点上运行，当节点从K8S集群中被移除，被DaemonSet调度的pod会被移除，如果删除DaemonSet，所有跟这个DaemonSet相关的pods都会被删除。

在某种程度上，DaemonSet承担了RC的部分功能，它也能保证相关pods持续运行，如果一个DaemonSet的Pod被杀死、停止、或者崩溃，那么DaemonSet将会重新创建一个新的副本在这台计算节点上。

一般应用于日志收集、监控采集、分布式存储守护进程、ingress等


### Job

从程序的运行形态上来区分，我们可以将Pod分为两类：长时运行服务（jboss、mysql等）和一次性任务（数据计算、测试）。RC创建的Pod都是长时运行的服务，而Job创建的Pod都是一次性任务。
  在Job的定义中，restartPolicy（重启策略）只能是Never和OnFailure。Job可以控制一次性任务的Pod的完成次数（Job-->spec-->completions）和并发执行数（Job-->spec-->parallelism），当Pod成功执行指定次数后，即认为Job执行完毕。
　　
### Cronjob

Kubernetes集群使用Cron Job管理基于时间的作业，可以在指定的时间点执行一次或在指定时间点执行多次任务。 一个Cron Job就好像Linux crontab中的一行，可以按照Cron定时运行任务

### StatefulSet

K8s在1.3版本里发布了Alpha版的PetSet功能。在云原生应用的体系里，有下面两组近义词；第一组是无状态（stateless）、牲畜（cattle）、无名（nameless）、可丢弃（disposable）；第二组是有状态（stateful）、宠物（pet）、有名（having name）、不可丢弃（non-disposable）。RC和RS主要是控制提供无状态服务的，其所控制的Pod的名字是随机设置的，一个Pod出故障了就被丢弃掉，在另一个地方重启一个新的Pod，名字变了、名字和启动在哪儿都不重要，重要的只是Pod总数；而PetSet是用来控制有状态服务，PetSet中的每个Pod的名字都是事先确定的，不能更改。PetSet中Pod的名字的作用，是用来关联与该Pod对应的状态。

　　对于RC和RS中的Pod，一般不挂载存储或者挂载共享存储，保存的是所有Pod共享的状态，Pod像牲畜一样没有分别；对于PetSet中的Pod，每个Pod挂载自己独立的存储，如果一个Pod出现故障，从其他节点启动一个同样名字的Pod，要挂在上原来Pod的存储继续以它的状态提供服务。

　　适合于PetSet的业务包括数据库服务MySQL和PostgreSQL，集群化管理服务Zookeeper、etcd等有状态服务。PetSet的另一种典型应用场景是作为一种比普通容器更稳定可靠的模拟虚拟机的机制。传统的虚拟机正是一种有状态的宠物，运维人员需要不断地维护它，容器刚开始流行时，我们用容器来模拟虚拟机使用，所有状态都保存在容器里，而这已被证明是非常不安全、不可靠的。使用PetSet，Pod仍然可以通过漂移到不同节点提供高可用，而存储也可以通过外挂的存储来提供高可靠性，PetSet做的只是将确定的Pod与确定的存储关联起来保证状态的连续性。

### Deployment
- 基于此强大特性在部署服务轻松执行[金丝雀发布](https://www.jevic.cn/2018/02/22/devops-deployment/)以及[蓝绿发布](https://www.jevic.cn/2018/02/22/devops-deployment/).....  强大到没有朋友👬 😄

Kubernetes提供了一种更加简单的更新RC和Pod的机制，叫做Deployment。通过在Deployment中描述你所期望的集群状态，Deployment Controller会将现在的集群状态在一个可控的速度下逐步更新成你所期望的集群状态。Deployment主要职责同样是为了保证pod的数量和健康，90%的功能与Replication Controller完全一样，可以看做新一代的Replication Controller。但是，它又具备了Replication Controller之外的新特性：

Deployment继承了上面描述的Replication Controller全部功能。

`事件和状态查看`：可以查看Deployment的升级详细进度和状态。

`回滚`：当升级pod镜像或者相关参数的时候发现问题，可以使用回滚操作回滚到上一个稳定的版本或者指定的版本。

`版本记录`: 每一次对Deployment的操作，都能保存下来，给予后续可能的回滚使用。

`暂停和启动`：对于每一次升级，都能够随时暂停和启动。

`多种升级方案`：
Recreate----删除所有已存在的pod,重新创建新的; 
RollingUpdate----滚动升级，逐步替换的策略，同时滚动升级时，支持更多的附加参数，例如设置最大不可用pod数量，最小升级间隔时间等等。
　　　


#### 弹性伸缩
```
kubectl scale [--resource-version=version] [--current-replicas=count] --replicas=COUNT (-f FILENAME | TYPE NAME)
```

缩小副本为1

```
kubectl scale --replicas=1 deployment flask-app-deploy  
```

扩容到5

```
kubectl scale --replicas=5 deployment flask-app-deploy  
```

#### 应用升级
>更新镜像 

```
1. kubectl set image deployment flask-app-deploy flask-app=jevic/app:v2
2. kubectl patch deployment flask-app-deploy -p '{"spec":{"containers":[{"name":"flask-app","image":"jevic/app:v2"}]}}'

暂停
kubectl rollout pause deployment flask-app-deploy 
取消暂停
kubectl pause resume deployment flask-app-deploy
```

#### 回滚

```
# kubectl rollout --help
# kubectl rollout history deployment flask-app-deploy
deployments "flask-app-deploy"
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
# kubectl get rs -o wide -l app=flask-app
NAME                          DESIRED   CURRENT   READY     AGE       CONTAINERS   IMAGES         SELECTOR
flask-app-deploy-5dcb69465c   3         3         3         6m        flask-app    jevic/app:v2   app=flask-app,pod-template-hash=1876250217
flask-app-deploy-6c4896f74b   0         0         0         9m        flask-app    jevic/app:v1   app=flask-app,pod-template-hash=2704529306
flask-app-deploy-9c5bc8bd8    0         0         0         13m       flask-app    jevic/app:v1   app=flask-app,pod-template-hash=571674684

# kubectl rollout undo deployment flask-app-deploy --to-revision=1
# kubectl get rs -o wide -l app=flask-app
NAME                          DESIRED   CURRENT   READY     AGE       CONTAINERS   IMAGES         SELECTOR
flask-app-deploy-5dcb69465c   0         0         0         9m        flask-app    jevic/app:v2   app=flask-app,pod-template-hash=1876250217
flask-app-deploy-6c4896f74b   0         0         0         12m       flask-app    jevic/app:v1   app=flask-app,pod-template-hash=2704529306
flask-app-deploy-9c5bc8bd8    3         3         3         15m       flask-app    jevic/app:v1   app=flask-app,pod-template-hash=571674684
```

--to-revision=1 回滚到指定版本,不指定默认回滚到前一个版本

## Services
 [Service ](https://kubernetes.io/docs/concepts/services-networking/service/)

在Kubernetes中，在受到RC调控的时候，Pod副本是变化的，对于的虚拟IP也是变化的，比如发生迁移或者伸缩的时候。这对于Pod的访问者来说是不可接受的。Kubernetes中的Service是一种抽象概念，它定义了一个Pod逻辑集合以及访问它们的策略，Service同Pod的关联同样是居于Label来完成的。Service的目标是提供一种桥梁， 它会为访问者提供一个固定访问地址，用于在访问时重定向到相应的后端，这使得非 Kubernetes原生应用程序，在无须为Kubemces编写特定代码的前提下，轻松访问后端。

Service同RC一样，都是通过Label来关联Pod的。当你在Service的yaml文件中定义了该Service的selector中的label为app:my-web，那么这个Service会将Pod-->metadata-->labeks中label为app:my-web的Pod作为分发请求的后端。当Pod发生变化时（增加、减少、重建等），Service会及时更新。这样一来，Service就可以作为Pod的访问入口，起到代理服务器的作用，而对于访问者来说，通过Service进行访问，无需直接感知Pod。

需要注意的是，Kubernetes分配给Service的固定IP是一个虚拟IP，并不是一个真实的IP，在外部是无法寻址的。真实的系统实现上，Kubernetes是通过Kube-proxy组件来实现的虚拟IP路由及转发。所以在之前集群部署的环节上，我们在每个Node上均部署了Proxy这个组件，从而实现了Kubernetes层级的虚拟转发网络。


### 工作模式
在kubernetes 的版本迭代历史到目前为止有三种工作模式:

- userspace: 1.1 版本之前使用
- iptables: 1.10 版本之前使用
- ipvs: 1.11 版本之后使用(如果ipvs 不可用自动降级使用iptables)

### 网络模式
- ClusterIP  集群内部通信
- NodePort 
    - 外部可访问
    - Client > NodeIP:NodePort > ClusterIP:svcPort > PodIP:containerPort
- LoadBalancer 负载均衡器
- ExternelName  FQDN
- None Headless Service
    - ServiceName > PodIP:containerPort 

### 创建 Services

- 使用 expose 

```
# kubectl expose deployment nginx-demo --name=nginx-demo-svc --port=80,443 
service "nginx-demo-svc" exposed
```

- 定义资源清单创建

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-demo
  labels:
    app: nginx-demo
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
    name: http
  - port: 443
    protocol: TCP
    name: https
  selector:
    app: nginx-demo
```

查看 services

```
# kubectl get svc -l nginx=nginx-demo
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
nginx-demo       ClusterIP   10.254.20.0     <none>        80/TCP,443/TCP   1d
nginx-demo-svc   ClusterIP   10.254.41.208   <none>        80/TCP,443/TCP   1m
```

- 解析服务

资源记录: SVC_NAME.NS_NAME.DOMAIN.LTD.

默认为: svc.cluster.local.

```
# kubectl get svc -n kube-system
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
coredns                ClusterIP   10.254.0.2       <none>        53/UDP,53/TCP   20d
# dig -t A nginx-demo.default.svc.cluster.local @10.254.0.2

; <<>> DiG 9.9.4-RedHat-9.9.4-29.el7 <<>> -t A nginx-demo.default.svc.cluster.local @10.254.0.2
....
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;nginx-demo.default.svc.cluster.local. IN A

;; ANSWER SECTION:
nginx-demo.default.svc.cluster.local. 5	IN A	10.254.20.0
....

# dig -t A nginx-demo-svc.default.svc.cluster.local @10.254.0.2

; <<>> DiG 9.9.4-RedHat-9.9.4-29.el7 <<>> -t A nginx-demo-svc.default.svc.cluster.local @10.254.0.2
....
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;nginx-demo-svc.default.svc.cluster.local. IN A

;; ANSWER SECTION:
nginx-demo-svc.default.svc.cluster.local. 5 IN A 10.254.41.208
.....
```

## ConfigMap
[configmap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)

很多生产环境中的应用程序配置较为复杂，可能需要多个config文件、命令行参数和环境变量的组合。并且，这些配置信息应该从应用程序镜像中解耦出来，以保证镜像的可移植性以及配置信息不被泄露。社区引入ConfigMap这个API资源来满足这一需求。　　

ConfigMap包含了一系列的键值对，用于存储被Pod或者系统组件（如controller）访问的信息。这与secret的设计理念有异曲同工之妙，它们的主要区别在于ConfigMap通常不用于存储敏感信息，而只存储简单的文本信息。

## Secret
[secret](https://kubernetes.io/zh/docs/concepts/configuration/secret/)

Secret 是一种包含少量敏感信息例如密码、token 或 key 的对象。这样的信息可能会被放在 Pod spec 中或者镜像中；将其放在一个 secret 对象中可以更好地控制它的用途，并降低意外暴露的风险。
用户可以创建 secret，同时系统也创建了一些 secret。
要使用 secret，pod 需要引用 secret。Pod 可以用两种方式使用 secret：作为 volume 中的文件被挂载到 pod 中的一个或者多个容器里，或者当 kubelet 为 pod 拉取镜像时使用。

## 证书
[证书](https://kubernetes.io/zh/docs/concepts/cluster-administration/certificates/)

当使用客户端证书进行认证时，用户可以使用现有部署脚本，或者通过 easyrsa、openssl 或 cfssl 手动生成证书。


