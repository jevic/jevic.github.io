---
layout: post
categories: Docker
title: Docker swarm 容器编排调度引擎demo
date: 2017-01-23 15:29:13 +0800
description: Docker swarm
keywords: docker swarm 
---


![这里写图片描述](http://www.zpedu.com/Admin/Umeditor/net/upload/2016-05-27/038c19fd-a770-4ad1-b6c5-7a88cfa87d49.png)
## 1. 环境准备:
#### 1.1 系统安装并初始化
* 操作系统环境
    * 操作系统安装此处不在叙述自行安装即可
    * 版本： CentOS Linux release 7.2.1511 (Core)
    * 三台 虚拟机
    * 关闭 `SELinux` 以及 `防火墙`
    * `注意`：安装时请选择时间同步(或手动安装开启ntp服务)
    * 更换 阿里云yum源 [ 分别在三台主机执行 ]

``` sh
[root@master ~]# wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
```

#### 1.2 IP 及主机名 配置
* 手动修改 /etc/hosts 

```sh
192.168.11.130  master
192.168.11.140  node01
192.168.11.150  node02

```

#### 1.3 配置ssh免密码登录
* 下面不在介绍直接给出执行步骤：

``` sh
[root@master ~]# ssh-keygen -t rsa    # 一路回车即可！
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
ef:79:38:51:85:ec:01:6b:62:28:2d:82:01:11:dd:37 root@master
The key's randomart image is:
+--[ RSA 2048]----+
|*+ .      .o .   |
| o. ..E.   .+ .  |
|. . o.o.o o. o   |
|   . o . o  o    |
|        S  .     |
|         ..      |
|          .o     |
|         .o..    |
|          oo     |
+-----------------+
1) 直接使用命令复制公钥
[root@master ~]# ssh-copy-id -i .ssh/id_rsa.pub root@node01
[root@master ~]# ssh-copy-id -i .ssh/id_rsa.pub root@node02

2) 某些情况下需要手动复制,下面以node01节点的操作为例：
[root@master ~]# scp .ssh/id_rsa.pub root@node01:/tmp
[root@node01 ~]# mkdir .ssh && chmod 700 .ssh
[root@node01 ~]# cat /tmp/id_rsa.pub >> .ssh/authorized_keys [此文件权限必须为600]    

[root@master ~]# ssh node01
首次执行会提示yes/no确认  之后不再提示！

```

## 2. 安装Docker以及初始配置:
#### 2.1 安装Docker [ 所有主机操作相同 ]

``` sh
1.也直接给出安装步骤：
[root@master ~]# tee /etc/yum.repos.d/docker.repo <<-'EOF'  
[dockerrepo]   
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF
[root@master ~]# yum install -y docker-engine

```
#### 2.2 查看Docker是否安装成功

``` sh
[root@master ~]# docker version
Client:
 Version:      1.12.4     # 客户端版本
 API version:  1.24
 Go version:   go1.6.4
 Git commit:   1564f02
 Built:        Mon Dec 12 23:41:49 2016
 OS/Arch:      linux/amd64

Server:
 Version:      1.12.4   # docker-engine 版本
 API version:  1.24
 Go version:   go1.6.4
 Git commit:   1564f02
 Built:        Mon Dec 12 23:41:49 2016
 OS/Arch:      linux/amd64
[root@master ~]# docker info
Containers: 0   容器数
 Running: 0   运行容器数
 Paused: 0   暂停
 Stopped: 0  停止
Images: 0  
Server Version: 1.12.4
Storage Driver: overlay   存储驱动
 Backing Filesystem: xfs  磁盘文件系统格式
Logging Driver: json-file
Cgroup Driver: cgroupfs
.......省略
Insecure Registries:
 jevic.io  私有仓库
 127.0.0.0/8

```

#### 2.3 修改docker的配置文件,并启动

``` sh
Docker daemon 配置参数：
[root@master ~]# dockerd --help
Usage: dockerd [OPTIONS]
A self-sufficient runtime for containers.
Options:
  --add-runtime=[]                         Register an additional OCI compatible runtime
  --api-cors-header                        Set CORS headers in the remote API
  --authorization-plugin=[]                Authorization plugins to load
  -b, --bridge          #指定容器使用的网络接口，默认为docker0，也可以指定其他网络接口
--bip                 #指定桥接地址，即定义一个容器的私有网络
--cgroup-parent       #为所有的容器指定父cgroup
--cluster-advertise   #为集群设定一个地址或者名字
--cluster-store       #后端分布式存储的URL
--cluster-store-opt=map[]  #设置集群存储参数
--config-file=/etc/docker/daemon.json  #指定配置文件
-D                    #启动debug模式
--default-gateway     #为容器设定默认的ipv4网关(--default-gateway-v6)
--dns=[]              #设置dns
--dns-opt=[]          #设置dns参数
--dns-search=[]       #设置dns域
--exec-opt=[]         #运行时附加参数
--exec-root=/var/run/docker  #设置运行状态文件存储目录
--fixed-cidr          #为ipv4子网绑定ip
-G, --group=docker    #设置docker运行时的属组
-g, --graph=/var/lib/docker  #设置docker运行时的家目录
-H, --host=[]         #设置docker程序启动后套接字连接地址
--icc=true            #是内部容器可以互相通信，环境中需要禁止内部容器访问
--insecure-registry=[] #设置内部私有注册中心地址
--ip=0.0.0.0          #当映射容器端口的时候默认的ip(这个应该是在多主机网络的时候会比较有用)
--ip-forward=true     #使net.ipv4.ip_forward生效，其实就是内核里面forward
--ip-masq=true        #启用ip伪装技术(容器访问外部程序默认不会暴露自己的ip)
--iptables=true       #启用容器使用iptables规则
-l, --log-level=info  #设置日志级别
--live-restore        #启用热启动(重启docker，保证容器一直运行1.12新特性)
--log-driver=json-file  #容器日志默认的驱动
--max-concurrent-downloads=3  #为每个pull设置最大并发下载
--max-concurrent-uploads=5    #为每个push设置最大并发上传
--mtu                   #设置容器网络的MTU
--oom-score-adjust=-500  #设置内存oom的平分策略(-1000/1000)
-p, --pidfile=/var/run/docker.pid  #指定pid所在位置
-s, --storage-driver     #设置docker存储驱动
--selinux-enabled        #启用selinux的支持
--storage-opt=[]         #存储参数驱动
--swarm-default-advertise-addr  #设置swarm默认的node节点
--tls                    #使用tls加密
--tlscacert=~/.docker/ca.pem  #配置tls CA 认证
--tlscert=~/.docker/cert.pem  #指定认证文件
--tlskey=~/.docker/key.pem    #指定认证keys
--userland-proxy=true         #为回环接口使用用户代理
--userns-remap                #为用户态的namespaces设定用户或组

[root@master ~]# grep 'ExecStart' /lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd -s=overlay \    修改存储驱动模式
--insecure-registry="jevic.io" \   添加私有仓库地址
--registry-mirror="http://df4b377c.m.daocloud.io"  添加Daocloud 镜像加速器
[root@master ~]# systemctl daemon-reload && systemctl start docker
[root@master ~]# systemctl enable docker 
```
## [Daocloud 镜像加速器](daocloud.io)

## 3. 创建swarm 集群
#### 3.1 初始化swarm集群并添加node节点

``` sh
[root@master ~]# docker swarm init --listen-addr 0.0.0.0
Swarm initialized: current node (73ju72f6nlyl9kiib7z5r0bsk) is now a manager.
To add a worker to this swarm, run the following command:
    docker swarm join \
    --token SWMTKN-1-3fwuvslxnympc94nyvvsuxm67j8ricvsbz4dez5qui835gul7d-5b4rylzx9ny8xa9v3sbmmum5b \
    192.168.11.130:2377
To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

查看worker token
[root@master ~]# docker swarm join-token worker
To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-3fwuvslxnympc94nyvvsuxm67j8ricvsbz4dez5qui835gul7d-5b4rylzx9ny8xa9v3sbmmum5b \
    192.168.11.130:2377

加入节点：
[root@node01 ~]# docker swarm join \
>     --token SWMTKN-1-3fwuvslxnympc94nyvvsuxm67j8ricvsbz4dez5qui835gul7d-5b4rylzx9ny8xa9v3sbmmum5b \
>     192.168.11.130:2377
[root@node02 ~]# docker swarm join \
>     --token SWMTKN-1-3fwuvslxnympc94nyvvsuxm67j8ricvsbz4dez5qui835gul7d-5b4rylzx9ny8xa9v3sbmmum5b \
>     192.168.11.130:2377

查看节点状态：
[root@master ~]# docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
a9a6l13yyqofyz4lq80liwdox    node02    Ready   Active        
czlqqxo2h8xqs1zmudgiawyj2 *  master    Ready   Active        Leader
exoa5djqpbjf4ded1n8wy9lz0    node01    Ready   Active 

```

####  添加新节点为 主节点
* 注：此处只提供操作步骤，本文并没有添加第二管理节点

``` sh
[root@master ~]# docker swarm join-token manager   管理节点token
To add a manager to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-3fwuvslxnympc94nyvvsuxm67j8ricvsbz4dez5qui835gul7d-2ia8ufgii6t1lfm33ix9fy32l \
    192.168.11.130:2377
[root@ ~]# docker swarm join \
>     --token SWMTKN-1-3fwuvslxnympc94nyvvsuxm67j8ricvsbz4dez5qui835gul7d-2ia8ufgii6t1lfm33ix9fy32l \
>     192.168.11.130:2377

再次查看主节点 node 状态 会发现多了一个 Reachable     
```

#### 3.2 创建overlay 网络

``` sh
[root@master ~]# docker network create --subnet=172.168.0.1/24 --driver overlay jevic-io
[root@master ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
de890f8e821a        bridge              bridge              local                              
9b9acc5bca76        host                host                local               
9rq59tuwqr75        ingress             overlay             swarm               
ev0rthinc6s2        jevic-io            overlay             swarm               
[root@master ~]# docker network inspect jevic-io
[
    {
        "Name": "jevic-io",
        "Id": "ev0rthinc6s2orc54nmxeqfa4",
        "Scope": "swarm",
        "Driver": "overlay",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.168.0.1/24",
                    "Gateway": "172.168.0.1"
                }
            ]
        },
        "Internal": false,
        "Containers": {
            "f8996ffc02d3c45e8b0f117953b66434bd29dc9d7b7020335b6747f70abf1f08": {
                "Name": "cadvisor.0.b5bsy9tkw51hm9fv1tkyffk58",
                "EndpointID": "1c9bd0b993290abd61a7b5310a598ecf469d574e80bb96847a61121564a0e729",
                "MacAddress": "02:42:ac:a8:00:08",
                "IPv4Address": "172.168.0.8/24",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.driver.overlay.vxlanid_list": "257"
        },
        "Labels": {}
    }
]
```
#### 3.3 运行测试容器
``` sh
[root@master ~]# docker service create \
--replicas 1 \  容器数量默认也为1,可以省略
--network jevic-io \
--name nginx \
-p 80:80 \
jevic.io/nginx:alpine
[root@master ~]# docker service ls
ID            NAME      REPLICAS  IMAGE                    COMMAND    
ar9q51stvbib  nginx     1/1       jevic.io/nginx:alpine    
[root@master ~]# docker service ps nginx
ID                         NAME         IMAGE                  NODE    DESIRED STATE  CURRENT STATE           
 ERROR9zxs8re9i3w0h5tppu4h94k6l  nginx.1      jevic.io/nginx:alpine  node02  Running        Running 10 minutes ago  
```


#### 3.4 扩展(Scaling)应用

``` sh
[root@master ~]# docker service scale nginx=3
[root@master ~]# docker service ps nginx
ID                         NAME         IMAGE                  NODE    DESIRED STATE  CURRENT STATE           
ERROR9zxs8re9i3w0h5tppu4h94k6l  nginx.1      jevic.io/nginx:alpine  node02  Running        Running about a minute ago
2cjk40jw5ps2cimjbnnk0uhz1  nginx.2      jevic.io/nginx:alpine  node01  Running        Running about a minute ago   
bx2m2lra1b1cj1ahhoaoe7ih4  nginx.3      jevic.io/nginx:alpine  node02  Running        Running about a minute ago
```

##### 指定Server服务的运行节点

``` sh
[root@master ~]# docker service create \ 
--network jevic-io \
--name tngx \
--constraint 'node.hostname==node02' \  只运行在node02 节点
-p 80:80 \
jevic.io/nginx:alpine
```

#### 3.5 测试docker swarm网络是否能互通
* 在主节点随意查看一个运行中的容器
* 在node节点进入容器，ping 主节点的容器名

``` sh
[root@master ~]# docker ps
CONTAINER ID        IMAGE                     COMMAND                  CREATED             STATUS             
 PORTS                   NAMESac9d6407f49f        jevic.io/nginx:alpine     "nginx -g 'daemon off"   24 minutes ago      Up 23 minutes      
 80/tcp, 443/tcp         web.1.dhfqiyk475yf37nbl19t0txdw
[root@node01 ~]# docker exec -it 79cf sh
# ping web.1.dhfqiyk475yf37nbl19t0txdw
PING web.1.dhfqiyk475yf37nbl19t0txdw (172.168.0.17) 56(84) bytes of data.
64 bytes from web.1.dhfqiyk475yf37nbl19t0txdw.jevic-io (172.168.0.17): icmp_seq=1 ttl=64 time=0.410 ms
64 bytes from web.1.dhfqiyk475yf37nbl19t0txdw.jevic-io (172.168.0.17): icmp_seq=2 ttl=64 time=0.187 ms
^C
--- web.1.dhfqiyk475yf37nbl19t0txdw ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.187/0.298/0.410/0.112 ms

```

#### 3.6 测试dokcer swarm自带的负载均衡

``` sh
[root@master ~]# docker service create --name app--mode global -p 8000:8000 jevic.io/nginx:alpine
1u87lrzlktgskt4g6ae30xzb8
[root@master ~]# docker service ps app
ID                         NAME        IMAGE           NODE  DESIRED STATE  CURRENT STATE           ERROR
cjf5w0pv5bbrph2gcvj508rvj  app     jevic.io/nginx:alpine  node01    Running        Running 16 minutes ago
dokh8j4z0iuslye0qa662axqv   \_ app jevic.io/nginx:alpine  node02    Running        Running 16 minutes ago
dumjwz4oqc5xobvjv9rosom0w   \_ app jevic.io/nginx:alpine  master    Running        Running 16 minutes ago

每次获取的值都不同：
[root@master ~]# curl $(hostname --all-ip-addresses | awk '{print $1}'):8000
I'm 8c2eeb5d420f
[root@master ~]# curl $(hostname --all-ip-addresses | awk '{print $1}'):8000
I'm 0b56c2a5b2a4
[root@master ~]# curl $(hostname --all-ip-addresses | awk '{print $1}'):8000
I'm 000982389fa0
```

## 扩展阅读链接
* [docker 官方文档](https://docs.docker.com/)
* [Docker 基础知识原理](http://www.open-open.com/lib/view/open1423703640748.html)
* [Docker 生态体系](http://dockone.io/article/205)
* [Docker 常见问题 F&Q 解答(新手必读)](http://blog.lab99.org/post/docker-2016-07-14-faq.html)
* [Docker Github](https://github.com/docker/docker/releases)


转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
