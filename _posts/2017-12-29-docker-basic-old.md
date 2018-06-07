---
layout: post
categories: Docker
title: docker 基础知识杂录
date: 2016-09-30 15:23:20 +0800
description: docker 基础知识杂录
keywords: docker
---


## Docker的主要优点：

* 轻量级资源使用：容器在进程级别隔离并使用宿主机的内核，而不需要虚拟化整个操作系统。

* 可移植性：一个容器应用所需要的依赖都在容器中，这就让它可以在任意一台Docker主机上运行。

* 可预测性：宿主机不需要关心容器内运行的是什么，同样，容器也不需要关心是在哪个宿主机上运行。所需要的接口都是标准化的，并且交互也都是可预测的。

* 通常在用Docker来设计应用或者服务时，最好的方法是打破面向服务架构的设计，而采用独立容器的设计。这让使用者在以后的使用中更容易的独立扩展或者升级组件。拥有如此的灵活性是人们对用Docker开发和部署感兴趣的原因之一。


## 服务发现
>etcd：服务发现/全局分布式键值对存储
consul：服务发现/全局分布式键值对存储
zookeeper：服务发现/全局分布式键值对存储
crypt：加密etcd条目的项目
confd：观测键值对存储变更和新值的触发器重新配置服务

## 调度和集群管理工具
>fleet: 调度器和集群管理工具
marathon:调度器和集群管理工具
Swarm:调度器和集群管理工具
mesos:宿主机抽象服务，用于为调度器联合宿主机资源
kubernetes:一个管理容器组的工具，具有先进的调度能力
compose:一个用于创建容器组的容器编排工具
 


## 内核升级 CentOS 7.2.1511 (Core)

``` sh
1.升级内核需要使用 elrepo 的yum 源首先我们导入 elrepo 的key
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org

2. 安装 elrepo 源
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm

3. 在yum的ELRepo源中，mainline 为最新版本的内核
yum --enablerepo=elrepo-kernel install  kernel-ml-devel kernel-ml -y

4. 修改内核启动顺序，默认启动的顺序应该为1,升级以后内核是往前面插入，为0    
grub2-set-default 0
5. 重启系统    
reboot
6. 查看 内核版本    
uname -r 

```

### install

``` sh
[root@master ~]# yum install -y yum-utils
[root@master ~]# yum-config-manager \
    --add-repo \
    https://docs.docker.com/engine/installation/linux/repo_files/centos/docker.repo
[root@master ~]# yum -y check-update
[root@master ~]# yum -y install docker-engine

加速器：
# curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://df4b377c.m.daocloud.io

[root@master ~]# systemctl daemon-reload && systemctl restart docker


开启守护进程[本地和网络]
# vim /usr/lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd -H unix:///var/run/docker.sock -H tcp://0.0.0.0:2375
# systemctl enable docker
# systemctl start docker
# docker version

```



### 基础操作(有些命令已经废弃不建议使用)

``` sh
[root@docker ~]# docker pull centos   下载镜像
[root@docker ~]# docker info
[root@docker ~]# docker version
[root@docker ~]# docker --help
[root@docker ~]# docker images  查看镜像
[root@docker ~]# docker search [images name]  搜索镜像
[root@docker ~]# docker history [images name]  查看镜像历史版本

管理命令
docker --help  


-i：交互式  -t: 打开一个终端 -d 后台运行 [不可同时使用] --rm 终止后立刻删除
[root@docker ~]# docker run [--rm] [-d] -i -t centos7 /bin/sh    进入docker镜像
[root@docker ~]# docker ps -l 
[root@docker ~]# docker stop id
[root@docker ~]# docker start id
[root@docker ~]# docker inspect mytest   【检查容器运行状态信息】


[root@docker ~]# docker stats testssh   【实时查看Container负载】


[root@docker ~]# docker logs -f mysql

[root@docker ~]# docker exec -it id /bin/bash  【exit退出后，容器依旧运行】

[root@docker ~]# docker rm id  删除容器

[root@docker ~]# docker rmi images_name 删除镜像
[root@docker ~]# docker rmi $(docker images -aq -f "dangling=true")  删除丢了标签的镜像

[root@docker ~]# docker ps -a | grep "Exited" | awk '{print $1 }'|xargs docker stop
[root@docker ~]# docker ps -a | grep "Exited" | awk '{print $1 }'|xargs docker rm


Docker提供了一个非常强大的命令diff，它可以列出容器内发生变化的文件和目录。
这些变化包括添加（A-add）、删除（D-delete）、修改（C-change）。
该命令便于Debug，并支持快速的共享环境。
[root@docker ~]# docker diff container

save 【持久化镜像】导入导出：
[root@docker ~]# docker save centos:7 > /home/save.tar
[root@docker ~]# docker load < /home/save.tar
export【持久化容器】导入导出：
[root@docker ~]# docker export [id] > [imagesname.tar]   导出镜像
[root@docker ~]# cat /home/export.tar | docker import - busybox-1-export:latest
远程路径导入：
[root@docker ~]# docker import http://example.com/test.tar

保存对容器的修改（commit）
当你对某一个容器做了修改之后（通过在容器中运行某一个命令），可以把对容器的修改保存下来，这样下次可以从保存后的最新状态运行该容器。
不建议使用此命令来构建镜像,里面的操作无法进行查看[黑箱操作]，构建镜像请使用Dockerfile

-a, --author="" Author; -m, --message="" Commit message
docker ps -l 查看ID
[root@docker ~]# docker commit ID new_image_name
Note:image相当于类，container相当于实例，不过可以动态给实例安装新软件，然后把这个container用commit命令固化成一个image


将容器中的文件拷贝到主机上
Usage: docker cp CONTAINER:PATH HOSTPATH
[root@docker ~]# docker cp testip:/tmp/testaa /tmp


-p:指定容器启动后docker上运行的端口映射，第一个为主机端口第二个为容器里的端口
[root@docker ~]# docker run -d -p 80:80 -p 8022:22 -p 2000-3000:2000-3000 lnmp

```



### docker数据卷

``` sh

创建一个数据卷挂载到容器的/webapp目录：
# docker run -it -d --name test -v /webapp docker.io/centos:7 /bin/bash
挂载一个主机目录作为数据卷[默认读写(rw),可以指定为只读(ro)]：
docker run -it --name=test1 -v /opt/webapp:/opt/webapp centos7:ip /bin/bash
挂载本地主机文件作为数据卷：[记录容器输入的历史]
docker run -rm -d /root/.bash_history:/.bash_history centos7 /bin/bash


数据卷容器：
1.> 创建数据卷容器：
# docker run -itd -v /mydata --name=volumes1 centos:6.8

2.> 创建容器，并从volumes1容器中挂载数据卷
# docker run -itd --volumes-from=volumes1 --volume-driver=/mydata --name votest1 centos:6.8
# docker run -itd --volumes-from=volumes1 --volume-driver=/mydata --name votest2 centos:6.8
3.>> 三个容器共享/mydata数据卷，都可以读写
注意：数据卷容器自身(volumes)并不需要保持运行状态！即使删除容器db1,db2，数据卷容器mydata里面的数据依旧存在；
[root@CentOS7 ~]# lsblk 
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda               8:0    0    8G  0 disk 
├─sda1            8:1    0  500M  0 part /boot
└─sda2            8:2    0  7.5G  0 part 
  ├─centos-root 253:0    0  6.7G  0 lvm  /
  └─centos-swap 253:1    0  820M  0 lvm  [SWAP]
sdb               8:16   0    8G  0 disk 
sr0              11:0    1 1024M  0 rom  
[root@CentOS7 ~]# lvs -a
  LV   VG     Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root centos -wi-ao----   6.67g                                                    
  swap centos -wi-ao---- 820.00m
  

容器互联：(废弃)
<在两个互联的容器之间创建了一个安全隧道，而且不用映射它们的端口到宿主机，避免暴漏数据库端口到外部网络上>
--link
docker run -d -P  --name=name --link container_name:alias images_name <Command>
docker run -d -P --name=web --link mysql:db centos:7 /bin/bash

```

### 存储驱动

* http://cloud.51cto.com/art/201412/461261.htm
* [Device-mapper 存储驱动](https://docs.docker.com/engine/userguide/storagedriver/device-mapper-driver/#image-layering-and-sharing
)

>联合文件系统（UnionFS）是一种分层、轻量级并且高性能的文件系统，它支持对文件系统的修改作为一次提交来一层层的叠加，
同时可以将不同目录挂载到同一个虚拟文件系统下(unite several directories into a single virtual filesystem)。
联合文件系统是 Docker 镜像的基础。镜像可以通过分层来进行继承，基于基础镜像（没有父镜像），可以制作各种具体的应用镜像。
另外，不同 Docker 容器就可以共享一些基础的文件系统层，同时再加上自己独有的改动层，大大提高了存储的效率。
Docker 中使用的 AUFS（AnotherUnionFS）就是一种联合文件系统。 
AUFS 支持为每一个成员目录（类似 Git 的分支）设定只读（readonly）、读写（readwrite）和写出（whiteout-able）权限, 同时 AUFS 里有一个类似分层的概念, 
对只读权限的分支可以逻辑上进行增量地修改(不影响只读部分的)。
Docker 目前支持的联合文件系统种类包括 AUFS, btrfs, vfs 和 DeviceMapper。



## Ubuntu

``` sh
升级内核：
$ sudo apt-get install -y --install-recommends linux-generic-lts-xenial
$ reboot

  
$ sudo apt-get update
E: Could not get lock /var/lib/apt/lists/lock - open (11: Resource temporarily unavailable)
E: Unable to lock the list directory
原因：刚装好的Ubantu系统，内部缺少很多软件源，这时，系统会自动启动软件源更新进程“apt-get”，并且它会一直存活。
由于它在运行时，会占用软件源更新时的系统锁（以下称“系统更新锁”，此锁文件在“/var/lib/apt/lists/”目录下），
而当有新的apt-get进程生成时，就会因为得不到系统更新锁而出现"E: 无法获得锁 /var/lib/apt/lists/lock - open (11: Resource temporarily unavailable)"错误提示！
因此，我们只要将原先的apt-get进程杀死，从新激活新的apt-get进程，就可以让软件管理器正常工作了！


安装docker
https://docs.docker.com/engine/installation/linux/ubuntu/ 

Ubuntu:
$ cat /etc/default/docker|egrep -v "^$|^#"
DOCKER_OPTS="-H unix:///var/run/docker.sock -H 0.0.0.0:2375"

```

## 扩展阅读 
- [Compose](https://docs.docker.com/compose/install/)
- https://docs.docker.com/compose/compose-file/#versioning
- https://docs.docker.com/compose/networking/#using-a-pre-existing-network
- [Swarm Mode](https://docs.docker.com/engine/swarm/)
- [Shipyard](https://shipyard-project.com/)


