---
layout: post
categories: Kubernetes
title: Kubeadm 快速部署kubernetes 1.10.1
date: 2018-07-17 20:35:46 +0800
description: Docker,kubernetes,k8s
keywords: Docker,k8s
---


## 系统环境

| IP地址 | 主机名 | Docker 版本 | kubernetes 版本 |
| --- | --- | --- | --- |
| 192.168.2.65 | k1.master | 17.03.1-ce | v1.10.1 |
| 192.168.2.66 | k2.master | 17.03.1-ce | v1.10.1 |
| 192.168.2.67 | k3.master | 17.03.1-ce | v1.10.1 |

### 初始化
>> 所有节点执行

```
[root@k1 ~]# setenforce 0
[root@k1 ~]# sed -i 's/SELINUX=enforcing/SELINUX=disalbe/g' /etc/sysconfig/selinux

[root@k1 ~]# cat >> /etc/security/limits.conf <<EOF
* soft nofile 65536
* hard nofile 65536
* soft nproc unlimited
* hard nproc unlimited
EOF

[root@k1 ~]# swapoff -a

[root@k1 ~]# cat >> /etc/sysctl.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=0
vm.swappiness=0
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF

[root@k1 ~]# systemctl stop firewalld && systemctl disable firewalld

[root@k1 ~]# wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

[root@k1 ~]# yum makecache fast
[root@k1 ~]# yum install -y epel-release
[root@k1 ~]# yum install -y conntrack ipvsadm ipset jq sysstat curl iptables libseccomp ntpdate
[root@k1 ~]# ntpdate cn.pool.ntp.org
```
#### 加载内核模块

```bash
# cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
ipvs_modules="ip_vs ip_vs_lc ip_vs_wlc ip_vs_rr ip_vs_wrr ip_vs_lblc ip_vs_lblcr ip_vs_dh ip_vs_sh ip_vs_fo ip_vs_nq ip_vs_sed ip_vs_ftp nf_conntrack_ipv4"
for kernel_module in \${ipvs_modules}; do
    /sbin/modinfo -F filename \${kernel_module} > /dev/null 2>&1
    if [ $? -eq 0 ]; then
        /sbin/modprobe \${kernel_module}
    fi
done
EOF
# chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep ip_vs
ip_vs_ftp              16384  0
ip_vs_sed              16384  0
ip_vs_nq               16384  0
ip_vs_fo               16384  0
ip_vs_sh               16384  0
ip_vs_dh               16384  0
ip_vs_lblcr            16384  0
ip_vs_lblc             16384  0
ip_vs_wrr              16384  0
ip_vs_rr               16384  0
ip_vs_wlc              16384  0
ip_vs_lc               16384  0
ip_vs                 147456  24 ip_vs_lblc,ip_vs_wrr,ip_vs_lc,ip_vs_lblcr,ip_vs_sed,ip_vs_ftp,ip_vs_wlc,ip_vs_rr,ip_vs_dh,ip_vs_sh,ip_vs_fo,ip_vs_nq
nf_nat                 28672  3 ip_vs_ftp,nf_nat_masquerade_ipv4,nf_nat_ipv4
nf_conntrack          106496  7 ip_vs,nf_conntrack_ipv4,nf_conntrack_netlink,nf_nat_masquerade_ipv4,xt_conntrack,nf_nat_ipv4,nf_nat
libcrc32c              16384  3 ip_vs,xfs,dm_persistent_data
```


#### Install Docker
- 参考 [官方文档](https://docs.docker.com/install/linux/docker-ce/centos/#install-using-the-repository)

```
[root@k1 ~]# yum install -y yum-utils device-mapper-persistent-data lvm2
[root@k1 ~]# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
[root@k1 ~]# yum list docker-ce --showduplicates | sort -r
[root@k1 ~]# yum -y install docker-ce-17.03.1.ce
[root@k1 ~]# systemctl start docker && systemctl stop docker
[root@k1 ~]# cat > /etc/docker/daemon.json <<EOF
{
  "registry-mirrors": ["https://dlvqhrac.mirror.aliyuncs.com"]
}
EOF
[root@k1 ~]# systemctl daemon-reload
[root@k1 ~]# systemctl start docker

```

##### 使用官方安装脚本自动安装

```
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```

##### 阿里云源站
-  使用[阿里云源](https://yq.aliyun.com/articles/110806)

```
# step 1: 安装必要的一些系统工具
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# Step 2: 添加软件源信息
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# Step 3: 更新并安装 Docker-CE
sudo yum makecache fast
sudo yum -y install docker-ce
# Step 4: 开启Docker服务
sudo service docker start

# 注意：
# 官方软件源默认启用了最新的软件，您可以通过编辑软件源的方式获取各个版本的软件包。例如官方并没有将测试版本的软件源置为可用，你可以通过以下方式开启。同理可以开启各种测试版本等。
# vim /etc/yum.repos.d/docker-ce.repo
#   将 [docker-ce-test] 下方的 enabled=0 修改为 enabled=1
#
# 安装指定版本的Docker-CE:
# Step 1: 查找Docker-CE的版本:
# yum list docker-ce.x86_64 --showduplicates | sort -r
#   Loading mirror speeds from cached hostfile
#   Loaded plugins: branch, fastestmirror, langpacks
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            docker-ce-stable
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            @docker-ce-stable
#   docker-ce.x86_64            17.03.0.ce-1.el7.centos            docker-ce-stable
#   Available Packages
# Step2 : 安装指定版本的Docker-CE: (VERSION 例如上面的 17.03.0.ce.1-1.el7.centos)
# sudo yum -y install docker-ce-[VERSION]
```

#### Kubeadm 组件
- 配置阿里云 yum 源

```
[root@k1 ~]# cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
enabled=1
EOF
```

- 安装组件

```
[root@k1 ~]# yum install kubelet kubeadm kubectl

```
- 所有节点配置kubectl 开机自启动

```
systemctl enable kubelet
```


## 初始化 Master
- 镜像
    - 出于不可描述的原因获取镜像会遇到困难
    - 使用开源社区镜像即可[详情查看](https://www.jevic.cn/2018/05/25/mirror/)
    - 具体操作细节在此忽略

- 修改配置

```
[root@k1 ~]# cat /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--fail-swap-on=false"
KUBE_PROXY_MODE=ipvs,ip_vs,ip_vs_rr,ip_vs_wrr,ip_vs_sh,nf_conntrack_ipv4

```

- 初始化

```
[root@k1 ~]# kubeadm init --kubernetes-version=v1.10.1 --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.2.65

记录最后的节点初始化信息，添加节点即可 
```
需要注意的是这里使用了flannel 网络插件所以`--pod-network-cidr`指定为了flannel默认的网段;
如果此次有更改记得在部署flannel时修改资源清单里面的网络配置!

- 添加flannel网络插件
	- https://github.com/coreos/flannel   

## 查看集群状态

```
[root@k1 ~]# kubectl get node
NAME        STATUS   ROLES    AGE     VERSION
k1.master   Ready    master   2d14h   v1.10.1
k2.master   Ready    <none>   2d14h   v1.10.1
k3.master   Ready    <none>   2d14h   v1.10.1

[root@k1 ~]# kubectl get pod -n kube-system
NAME                                READY   STATUS             RESTARTS   AGE
etcd-k1.master                      1/1     Running            2          2d14h
kube-apiserver-k1.master            1/1     Running            5          2d14h
kube-controller-manager-k1.master   1/1     Running            4          2d14h
kube-flannel-ds-amd64-9xzfl         1/1     Running            2          2d14h
kube-flannel-ds-amd64-krc4l         1/1     Running            1          2d14h
kube-flannel-ds-amd64-phhc7         1/1     Running            2          2d14h
kube-proxy-4vndk                    1/1     Running            3          2d14h
kube-proxy-f4dqb                    1/1     Running            2          2d14h
kube-proxy-s6p9z                    1/1     Running            2          2d14h
kube-scheduler-k1.master            1/1     Running            4          2d14h

```


转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
