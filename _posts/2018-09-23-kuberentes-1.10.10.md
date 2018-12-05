---
layout: post
categories: Kubernetes
title: 二进制手动部署kubernetes 1.10.10
date: 2018-09-23 20:35:46 +0800
description: Docker,kubernetes,k8s
keywords: Docker,k8s
---


>通读一遍先熟悉了过程在实际操作!!!

>通读一遍先熟悉了过程在实际操作!!!

>通读一遍先熟悉了过程在实际操作!!!


>关于镜像请查看最后的补充说明

>部署脚本查看[仓库脚本](https://github.com/jevic/kshell.git)

## 一. 系统环境
> CentOS Linux release 7.2.1511 (Core)


| IP地址 | 主机名 | Docker 版本 | kubernetes 版本 |
| --- | --- | --- | --- |
| 192.168.2.219 | master219 | 18.06.0-ce | v1.10.10 |
| 192.168.2.220 | node220 | 18.06.0-ce | v1.10.10 |
| 192.168.2.221 | node221 | 18.06.0-ce | v1.10.10 |

### 初始化
#### 1.1 配置ssh 

```
ssh-keygen -t rsa
一路回车....

将公钥添加到每个节点
```

- 批量执行脚本:

```
[root@master219 bin]# pwd
/usr/local/bin
[root@master219 bin]# cat cmd
#!/bin/bash
## 批量执行命令
iplist="node220 node221"

for i in $iplist
do
   echo -e "\033[32mssh $i \"$*\"\033[0m"
   ssh $i "$*"
done
[root@master219 bin]# cat rsy
#!/bin/bash
## 批量同步文件
iplist="node220 node221"

for i in $iplist
do
   echo -e "\033[32mscp -r $1 $i:$2\033[0m"
   scp -r $1 $i:$2
done

```


#### 1.2 系统配置
>> 所有节点都需要执行下列操作

```
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disalbe/g' /etc/sysconfig/selinux

cat >> /etc/security/limits.conf <<EOF
* soft nofile 65536
* hard nofile 65536
* soft nproc unlimited
* hard nproc unlimited
EOF

swapoff -a

cat >> /etc/sysctl.conf <<EOF
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

systemctl stop firewalld && systemctl disable firewalld
yum makecache fast
yum install -y epel-release
yum install -y conntrack ipvsadm ipset jq sysstat curl iptables libseccomp ntpdate
ntpdate cn.pool.ntp.org
```

#### 1.3 Install Docker
- 参考 [官方文档](https://docs.docker.com/install/linux/docker-ce/centos/#install-using-the-repository)

```
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum list docker-ce --showduplicates | sort -r
yum -y install docker-ce-18.06.0.ce
systemctl start docker && systemctl stop docker
cat > /etc/docker/daemon.json <<EOF
{
  "registry-mirrors": ["https://dlvqhrac.mirror.aliyuncs.com"]
}
EOF

systemctl daemon-reload
systemctl start docker
systemctl enable docker
```
## 二. 安装cfssl
- https://github.com/cloudflare/cfssl/releases

只需要在master节点安装使用即可，后续证书同步到其他节点即可

```
tar -zxvf cfssl.tar.gz
mv cfssl cfssljson /usr/local/bin
chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson
rm -f cfssl.tar.gz
```

## 三. ETCD
- [下载链接](https://github.com/coreos/etcd/releases/download)

> 创建etcd 用户来启动!! 注意用户权限

### 3.1 证书
- etcd-csr.json

```
{
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "etcd",
      "OU": "etcd Security",
      "L": "Shengzhen",
      "ST": "Shengzhen",
      "C": "CN"
    }
  ],
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "localhost",
    "192.168.2.219",
    "192.168.2.220",
    "192.168.2.221"
  ]
}
```

- etcd-gencert.json

```
{
  "signing": {
    "default": {
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ],
        "expiry": "87600h"
    }
  }
}
```

- etcd-root-ca-csr.json

```
{
  "key": {
    "algo": "rsa",
    "size": 4096
  },
  "names": [
    {
      "O": "etcd",
      "OU": "etcd Security",
      "L": "Shengzhen",
      "ST": "Shengzhen",
      "C": "CN"
    }
  ],
  "CN": "etcd-root-ca"
}
```

- 生成证书

```
cfssl gencert --initca=true etcd-root-ca-csr.json | cfssljson --bare etcd-root-ca
cfssl gencert --ca etcd-root-ca.pem --ca-key etcd-root-ca-key.pem --config etcd-gencert.json etcd-csr.json | cfssljson --bare etcd

[root@master219 ssl]# ls
etcd.csr  etcd-csr.json  etcd-gencert.json  etcd-key.pem  etcd.pem  etcd-root-ca.csr  etcd-root-ca-csr.json  etcd-root-ca-key.pem  etcd-root-ca.pem
```

### 3.2 etcd.conf

```
# [member]
ETCD_NAME=etcd1
ETCD_DATA_DIR="/var/lib/etcd/etcd1.etcd"
ETCD_WAL_DIR="/var/lib/etcd/wal"
ETCD_SNAPSHOT_COUNT="100"
ETCD_HEARTBEAT_INTERVAL="100"
ETCD_ELECTION_TIMEOUT="1000"
ETCD_LISTEN_PEER_URLS="https://192.168.2.219:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.2.219:2379,http://127.0.0.1:2379"
ETCD_MAX_SNAPSHOTS="5"
ETCD_MAX_WALS="5"
#ETCD_CORS=""
# [cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.2.219:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.2.219:2379"
# if you use different ETCD_NAME (e.g. test), set ETCD_INITIAL_CLUSTER value for this name, i.e. "test=http://..."
ETCD_INITIAL_CLUSTER="etcd1=https://192.168.2.219:2380,etcd2=https://192.168.2.220:2380,etcd3=https://192.168.2.221:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
#ETCD_DISCOVERY=""
#ETCD_DISCOVERY_SRV=""
#ETCD_DISCOVERY_FALLBACK="proxy"
#ETCD_DISCOVERY_PROXY=""
#ETCD_STRICT_RECONFIG_CHECK="false"
#ETCD_AUTO_COMPACTION_RETENTION="0"
# [proxy]
#ETCD_PROXY="off"
#ETCD_PROXY_FAILURE_WAIT="5000"
#ETCD_PROXY_REFRESH_INTERVAL="30000"
#ETCD_PROXY_DIAL_TIMEOUT="1000"
#ETCD_PROXY_WRITE_TIMEOUT="5000"
#ETCD_PROXY_READ_TIMEOUT="0"
# [security]
ETCD_CERT_FILE="/etc/etcd/ssl/etcd.pem"
ETCD_KEY_FILE="/etc/etcd/ssl/etcd-key.pem"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/ssl/etcd-root-ca.pem"
ETCD_AUTO_TLS="true"
ETCD_PEER_CERT_FILE="/etc/etcd/ssl/etcd.pem"
ETCD_PEER_KEY_FILE="/etc/etcd/ssl/etcd-key.pem"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/ssl/etcd-root-ca.pem"
ETCD_PEER_AUTO_TLS="true"
# [logging]
#ETCD_DEBUG="false"
# examples for -log-package-levels etcdserver=WARNING,security=DEBUG
#ETCD_LOG_PACKAGE_LEVELS=""
```

### 3.3 etcd.service

```
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
User=etcd
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/local/bin/etcd --name=\"${ETCD_NAME}\" --data-dir=\"${ETCD_DATA_DIR}\" --listen-client-urls=\"${ETCD_LISTEN_CLIENT_URLS}\""
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
```

### 3.4 安装启动 
- install.sh

```
#!/bin/bash
set -e
ETCD_VERSION="3.2.18"

preinstall(){
    getent group etcd >/dev/null || groupadd -r etcd
    getent passwd etcd >/dev/null || useradd -r -g etcd -d /var/lib/etcd -s /sbin/nologin -c "etcd user" etcd
}

install(){
    echo -e "\033[32mINFO: Copy etcd...\033[0m"
    tar -zxvf etcd-v${ETCD_VERSION}-linux-amd64.tar.gz
    cp etcd-v${ETCD_VERSION}-linux-amd64/etcd* /usr/local/bin
    rm -rf etcd-v${ETCD_VERSION}-linux-amd64
    echo -e "\033[32mINFO: Copy etcd config...\033[0m"
    cp -r conf /etc/etcd
    chown -R etcd:etcd /etc/etcd
    chmod -R 755 /etc/etcd/ssl
    echo -e "\033[32mINFO: Copy etcd systemd config...\033[0m"
    cp etcd.service /lib/systemd/system
    systemctl daemon-reload
}

postinstall(){
    if [ ! -d "/var/lib/etcd" ]; then
        mkdir /var/lib/etcd
        chown -R etcd:etcd /var/lib/etcd
    fi
}

preinstall
install
postinstall

```

目录结构:

```
etcd/
├── conf
│   ├── etcd.conf
│   └── ssl
│       ├── etcd.csr
│       ├── etcd-csr.json
│       ├── etcd-gencert.json
│       ├── etcd-key.pem
│       ├── etcd.pem
│       ├── etcd-root-ca.csr
│       ├── etcd-root-ca-csr.json
│       ├── etcd-root-ca-key.pem
│       └── etcd-root-ca.pem
├── etcd.service
├── etcd-v3.2.18-linux-amd64.tar.gz
└── install.sh
```

创建对应目录以及相关文件，保存 install.sh 脚本;将整个目录同步到各个节点执行即可；
然后在各个节点修改对应的etcd 名称及IP即可；

- 注意:三个节点同时执行此操作否则单个启动会卡死在那里;
- 使用前面配置的`cmd` 命令批量执行

```
# systemctl start etcd
# cmd systemctl enable etcd
```

```
export ETCDCTL_API=3
etcdctl --cacert=/etc/etcd/ssl/etcd-root-ca.pem --cert=/etc/etcd/ssl/etcd.pem --key=/etc/etcd/ssl/etcd-key.pem --endpoints=https://192.168.2.219:2379,https://192.168.2.220:2379,https://192.168.2.221:2379 endpoint health

https://192.168.2.221:2379 is healthy: successfully committed proposal: took = 2.077429ms
https://192.168.2.219:2379 is healthy: successfully committed proposal: took = 1.421477ms
https://192.168.2.220:2379 is healthy: successfully committed proposal: took = 2.222464ms
```


## 四. kubernetes
- 基于 hyperkube 二进制手动安装
- hyperkube是一个集成二进制运行文件
- 可以使用“hyperkube kubelet …”来启动kubelet ，用“hyperkube apiserver …”运行apiserver 等等。

### 4.1 证书
由于 kubelet 和 kube-proxy 用到的 kubeconfig 配置文件需要借助 kubectl 来生成，所以需要先安装一下 kubectl

```
curl https://storage.googleapis.com/kubernetes-release/release/v1.10.10/bin/linux/amd64/hyperkube -o hyperkube-1.10.10
chmod +x hyperkube_1.10.10
cp hyperkube_1.10.10 /usr/local/bin/hyperkube
ln -s /usr/local/bin/hyperkube /usr/local/bin/kubectl
```

[curl 代理配置](https://www.jevic.cn/2018/03/19/shadowsock/)

#### admin-csr.json

```
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Shengzhen",
      "L": "Shengzhen",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
```

#### k8s-gencert.json

```
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
```

#### k8s-root-ca-csr.json

```
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 4096
  },
  "names": [
    {
      "C": "CN",
      "ST": "Shengzhen",
      "L": "Shengzhen",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

#### kube-apiserver-csr.json

```
{
    "CN": "kubernetes",
    "hosts": [
        "127.0.0.1",
        "10.254.0.1",
        "192.168.2.219",
        "192.168.2.220",
        "192.168.2.221",
        "*.kubernetes.master",
        "localhost",
        "kubernetes",
        "kubernetes.default",
        "kubernetes.default.svc",
        "kubernetes.default.svc.cluster",
        "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Shengzhen",
            "L": "Shengzhen",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
```

#### kube-proxy-csr.json

```
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Shengzhen",
      "L": "Shengzhen",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

#### 4.1.1 生成证书和配置

```
# 生成 CA
cfssl gencert --initca=true k8s-root-ca-csr.json | cfssljson --bare k8s-root-ca

# 依次生成其他组件证书
for targetName in kube-apiserver admin kube-proxy; do
    cfssl gencert --ca k8s-root-ca.pem --ca-key k8s-root-ca-key.pem --config k8s-gencert.json --profile kubernetes $targetName-csr.json | cfssljson --bare $targetName
done

# 地址默认为 127.0.0.1:6443
# 如果在 master 上启用 kubelet 请在生成后的 kubeconfig 中
# 修改该地址为 当前MASTER_IP:6443
KUBE_APISERVER="https://127.0.0.1:6443"
BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
echo "Tokne: ${BOOTSTRAP_TOKEN}"

# 不要质疑 system:bootstrappers 用户组是否写错了，有疑问请参考官方文档
# https://kubernetes.io/docs/admin/kubelet-tls-bootstrapping/
cat > token.csv <<EOF
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:bootstrappers"
EOF

echo "Create kubelet bootstrapping kubeconfig..."
# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=k8s-root-ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=bootstrap.kubeconfig
# 设置客户端认证参数
kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig
# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig
# 设置默认上下文
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig

echo "Create kube-proxy kubeconfig..."
# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=k8s-root-ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig
# 设置客户端认证参数
kubectl config set-credentials kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
# 设置默认上下文
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

# 创建高级审计配置
cat >> audit-policy.yaml <<EOF
# Log all requests at the Metadata level.
apiVersion: audit.k8s.io/v1beta1
kind: Policy
rules:
- level: Metadata
EOF
```

生成后的文件

```
ssl
├── admin.csr
├── admin-csr.json
├── admin-key.pem
├── admin.pem
├── audit-policy.yaml
├── bootstrap.kubeconfig
├── genconfig.sh
├── k8s-gencert.json
├── k8s-root-ca.csr
├── k8s-root-ca-csr.json
├── k8s-root-ca-key.pem
├── k8s-root-ca.pem
├── kube-apiserver.csr
├── kube-apiserver-csr.json
├── kube-apiserver-key.pem
├── kube-apiserver.pem
├── kube-proxy.csr
├── kube-proxy-csr.json
├── kube-proxy-key.pem
├── kube-proxy.kubeconfig
├── kube-proxy.pem
└── token.csv
```

### 4.2 systemd 配置

#### kube-apiserver.service

```
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target
After=etcd.service
[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/apiserver
User=root
ExecStart=/usr/local/bin/hyperkube apiserver \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_ETCD_SERVERS \
            $KUBE_API_ADDRESS \
            $KUBE_API_PORT \
            $KUBELET_PORT \
            $KUBE_ALLOW_PRIV \
            $KUBE_SERVICE_ADDRESSES \
            $KUBE_ADMISSION_CONTROL \
            $KUBE_API_ARGS
Restart=on-failure
Type=notify
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
```

#### kube-controller-manager.service
```
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/controller-manager
User=root
ExecStart=/usr/local/bin/hyperkube controller-manager \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_MASTER \
            $KUBE_CONTROLLER_MANAGER_ARGS
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
```

#### kubelet.service
```
[Unit]
Description=Kubernetes Kubelet Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service
[Service]
WorkingDirectory=/var/lib/kubelet
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/kubelet
ExecStart=/usr/local/bin/hyperkube kubelet \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBELET_API_SERVER \
            $KUBELET_ADDRESS \
            $KUBELET_PORT \
            $KUBELET_HOSTNAME \
            $KUBE_ALLOW_PRIV \
            $KUBELET_ARGS
Restart=on-failure
KillMode=process
[Install]
WantedBy=multi-user.target
```

#### kube-proxy.service
```
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target
[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/proxy
ExecStart=/usr/local/bin/hyperkube proxy \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_MASTER \
            $KUBE_PROXY_ARGS
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
```

#### kube-scheduler.service
```
[Unit]
Description=Kubernetes Scheduler Plugin
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/scheduler
User=root
ExecStart=/usr/local/bin/hyperkube scheduler \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_MASTER \
            $KUBE_SCHEDULER_ARGS
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
```
### 4.3 配置文件
#### 4.3.1 master 节点配置
Master 节点主要会运行 3 各组件: kube-apiserver、kube-controller-manager、kube-scheduler，其中用到的配置文件如下
##### config
config 是一个通用配置文件，值得注意的是由于安装时对于 Node、Master 节点都会包含该文件;
在 Node 节点上请注释掉 KUBE_MASTER 变量，因为 Node 节点需要做 HA，要连接本地的 6443 加密端口；而这个变量将会覆盖 kubeconfig 中指定的 127.0.0.1:6443 地址

```
###
# kubernetes system config
#
# The following values are used to configure various aspects of all
# kubernetes services, including
#
#   kube-apiserver.service
#   kube-controller-manager.service
#   kube-scheduler.service
#   kubelet.service
#   kube-proxy.service
# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=2"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=true"

# How the controller-manager, scheduler, and proxy find the apiserver
KUBE_MASTER="--master=http://127.0.0.1:8080"
```

##### apiserver
apiserver 配置相对于 1.8 略有变动，其中准入控制器(admission control)选项名称变为了 --enable-admission-plugins，控制器列表也有相应变化，这里采用官方推荐配置，具体请参考 [官方文档](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#is-there-a-recommended-set-of-admission-controllers-to-use)

```
###
# kubernetes system config
#
# The following values are used to configure the kube-apiserver
#
# The address on the local server to listen to.
KUBE_API_ADDRESS="--advertise-address=192.168.2.219 --bind-address=192.168.2.219"
# The port on the local server to listen on.
KUBE_API_PORT="--secure-port=6443"
# Port minions listen on
# KUBELET_PORT="--kubelet-port=10250"
# Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd-servers=https://192.168.2.219:2379,https://192.168.2.220:2379,https://192.168.2.221:2379"
# Address range to use for services
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
# default admission control policies
KUBE_ADMISSION_CONTROL="--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota,NodeRestriction"
# Add your own!
KUBE_API_ARGS=" --anonymous-auth=false \
                --apiserver-count=3 \
                --audit-log-maxage=30 \
                --audit-log-maxbackup=3 \
                --audit-log-maxsize=100 \
                --audit-log-path=/var/log/kube-audit/audit.log \
                --audit-policy-file=/etc/kubernetes/audit-policy.yaml \
                --authorization-mode=Node,RBAC \
                --client-ca-file=/etc/kubernetes/ssl/k8s-root-ca.pem \
                --enable-bootstrap-token-auth \
                --enable-garbage-collector \
                --enable-logs-handler \
                --enable-swagger-ui \
                --etcd-cafile=/etc/etcd/ssl/etcd-root-ca.pem \
                --etcd-certfile=/etc/etcd/ssl/etcd.pem \
                --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem \
                --etcd-compaction-interval=5m0s \
                --etcd-count-metric-poll-period=1m0s \
                --event-ttl=48h0m0s \
                --kubelet-https=true \
                --kubelet-timeout=3s \
                --log-flush-frequency=5s \
                --token-auth-file=/etc/kubernetes/token.csv \
                --tls-cert-file=/etc/kubernetes/ssl/kube-apiserver.pem \
                --tls-private-key-file=/etc/kubernetes/ssl/kube-apiserver-key.pem \
                --service-node-port-range=30000-50000 \
                --service-account-key-file=/etc/kubernetes/ssl/k8s-root-ca.pem \
                --storage-backend=etcd3 \
                --enable-swagger-ui=true"
```

##### controller-manager
controller manager 配置默认开启了证书轮换能力用于自动签署 kueblet 证书，并且证书时间也设置了 10 年，可自行调整；增加了 --controllers 选项以指定开启全部控制器

```
###
# The following values are used to configure the kubernetes controller-manager
# defaults from config and apiserver should be adequate
# Add your own!
KUBE_CONTROLLER_MANAGER_ARGS="  --bind-address=0.0.0.0 \
                                --cluster-name=kubernetes \
                                --cluster-signing-cert-file=/etc/kubernetes/ssl/k8s-root-ca.pem \
                                --cluster-signing-key-file=/etc/kubernetes/ssl/k8s-root-ca-key.pem \
                                --controllers=*,bootstrapsigner,tokencleaner \
                                --deployment-controller-sync-period=10s \
                                --experimental-cluster-signing-duration=86700h0m0s \
                                --leader-elect=true \
                                --node-monitor-grace-period=40s \
                                --node-monitor-period=5s \
                                --pod-eviction-timeout=5m0s \
                                --terminated-pod-gc-threshold=50 \
                                --root-ca-file=/etc/kubernetes/ssl/k8s-root-ca.pem \
                                --service-account-private-key-file=/etc/kubernetes/ssl/k8s-root-ca-key.pem \
                                --feature-gates=RotateKubeletServerCertificate=true"
```

##### scheduler
```
###
# kubernetes scheduler config

# default config should be adequate

# Add your own!
KUBE_SCHEDULER_ARGS="   --address=0.0.0.0 \
                        --leader-elect=true \
                        --algorithm-provider=DefaultProvider"
```

#### 4.3.2 node 节点配置
Node 节点上主要有 kubelet、kube-proxy 组件，用到的配置如下

##### kubelet
kubeket 默认也开启了证书轮换能力以保证自动续签相关证书，同时增加了 --node-labels 选项为 node 打一个标签，关于这个标签最后部分会有讨论，如果在 master 上启动 kubelet，请将 node-role.kubernetes.io/k8s-node=true 修改为 node-role.kubernetes.io/k8s-master=true

```
###
# kubernetes kubelet (minion) config
# The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--node-ip=192.168.2.219"
# The port for the info server to serve on
# KUBELET_PORT="--port=10250"
# You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname-override=master219"
# location of the api-server
# KUBELET_API_SERVER=""
# Add your own!
KUBELET_ARGS="  --bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \
                --cert-dir=/etc/kubernetes/ssl \
                --cgroup-driver=cgroupfs \
                --cluster-dns=10.254.0.2 \
                --cluster-domain=cluster.local. \
                --fail-swap-on=false \
                --feature-gates=RotateKubeletClientCertificate=true,RotateKubeletServerCertificate=true \
                --node-labels=node-role.kubernetes.io/master=true \
                --image-gc-high-threshold=70 \
                --image-gc-low-threshold=50 \
                --kube-reserved=cpu=500m,memory=512Mi,ephemeral-storage=1Gi \
                --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
                --system-reserved=cpu=1000m,memory=1024Mi,ephemeral-storage=1Gi \
                --serialize-image-pulls=false \
                --sync-frequency=30s \
                --pod-infra-container-image=k8s.gcr.io/pause-amd64:3.0 \
                --resolv-conf=/etc/resolv.conf \
                --rotate-certificates"
```

##### proxy
```
###
# kubernetes proxy config
# default config should be adequate
# Add your own!
KUBE_PROXY_ARGS="--bind-address=0.0.0.0 \
                 --hostname-override=node219 \
                 --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \
                 --cluster-cidr=10.254.0.0/16"
```

### 4.4 启动 Master 节点 
创建相关目录否则报错
```
mkdir /var/log/kube-audit
mkdir /var/lib/kubelet
mkdir /usr/libexec
```


```
cp -a conf /etc/kubernetes  
cp systemd/*.service /lib/systemd/system
systemctl daemon-reload
```

>对于 `master` 节点启动无需做过多处理，多个 `master` 只要保证 `apiserver` 等配置中的 `ip` 地址监听没问题后直接启动即可

```
systemctl daemon-reload
systemctl start kube-apiserver
systemctl start kube-controller-manager
systemctl start kube-scheduler
systemctl enable kube-apiserver
systemctl enable kube-controller-manager
systemctl enable kube-scheduler

```

启动后:
![](https://raw.githubusercontent.com/jevic/images/master/kubernetes/kubernetes-get-cs.png)

### 4.5 启动 Node 节点
>由于 HA 等功能需要，对于 Node 需要做一些处理才能启动，主要有以下两个地方需要处理



#### 4.5.1 nginx-proxy (node节点部署)
在启动 kubelet、kube-proxy 服务之前，需要在本地启动 nginx 来 tcp 负载均衡 apiserver 6443 端口，nginx-proxy 使用 docker + systemd 启动，配置如下

- nginx-proxy.service

```
[Unit]
Description=kubernetes apiserver docker wrapper
Wants=docker.socket
After=docker.service

[Service]
User=root
PermissionsStartOnly=true
ExecStart=/usr/bin/docker run -p 127.0.0.1:6443:6443 \
                              -v /etc/nginx:/etc/nginx \
                              --name nginx-proxy \
                              --net=host \
                              --restart=on-failure:5 \
                              --memory=512M \
                              nginx:1.13.12-alpine
ExecStartPre=-/usr/bin/docker rm -f nginx-proxy
ExecStop=/usr/bin/docker stop nginx-proxy
Restart=always
RestartSec=15s
TimeoutStartSec=30s

[Install]
WantedBy=multi-user.target

```

- nginx.conf

```
error_log stderr notice;

worker_processes auto;
events {
        multi_accept on;
        use epoll;
        worker_connections 1024;
}

stream {
    upstream kube_apiserver {
        least_conn;
        server 192.168.2.219:6443; ## 多个master 依次配置即可
        # server x.x.x.x:6443;
    }

    server {
        listen        0.0.0.0:6443;
        proxy_pass    kube_apiserver;
        proxy_timeout 10m;
        proxy_connect_timeout 1s;
    }
}

```

启动 apiserver 的本地负载均衡

```
mkdir /etc/nginx
cp nginx.conf /etc/nginx
cp nginx-proxy.service /lib/systemd/system

systemctl daemon-reload
systemctl start nginx-proxy
systemctl enable nginx-proxy

```

#### 4.5.2 TLS bootstrapping
创建好 nginx-proxy 后不要忘记为 TLS Bootstrap 创建相应的 RBAC 规则，这些规则能实现证自动签署 TLS Bootstrap 发出的 CSR 请求，从而实现证书轮换(创建一次即可);

- tls-bootstrapping-clusterrole.yaml

```
# A ClusterRole which instructs the CSR approver to approve a node requesting a
# serving cert matching its client cert.
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeserver
rules:
- apiGroups: ["certificates.k8s.io"]
  resources: ["certificatesigningrequests/selfnodeserver"]
  verbs: ["create"]
```

在 master 执行创建

```
# 给与 kubelet-bootstrap 用户进行 node-bootstrapper 的权限
kubectl create clusterrolebinding kubelet-bootstrap \
    --clusterrole=system:node-bootstrapper \
    --user=kubelet-bootstrap

kubectl create -f tls-bootstrapping-clusterrole.yaml

# 自动批准 system:bootstrappers 组用户 TLS bootstrapping 首次申请证书的 CSR 请求
kubectl create clusterrolebinding node-client-auto-approve-csr \
        --clusterrole=system:certificates.k8s.io:certificatesigningrequests:nodeclient \
        --group=system:bootstrappers

# 自动批准 system:nodes 组用户更新 kubelet 自身与 apiserver 通讯证书的 CSR 请求
kubectl create clusterrolebinding node-client-auto-renew-crt \
        --clusterrole=system:certificates.k8s.io:certificatesigningrequests:selfnodeclient \
        --group=system:nodes

# 自动批准 system:nodes 组用户更新 kubelet 10250 api 端口证书的 CSR 请求
kubectl create clusterrolebinding node-server-auto-renew-crt \
        --clusterrole=system:certificates.k8s.io:certificatesigningrequests:selfnodeserver \
        --group=system:nodes

```


#### 4.5.3 启动
多节点部署时先启动好 nginx-proxy，然后修改好相应配置的 ip 地址等配置，最终直接启动即可(master 上启动 kubelet 不要忘了修改 kubeconfig 中的 apiserver 地址，还有对应的 kubelet 的 node label)


```
systemctl daemon-reload
systemctl start kubelet
systemctl start kube-proxy
systemctl enable kubelet
systemctl enable kube-proxy
```


#### 4.5.4 master 节点启动 kubelet
>注意: 对于在 master 节点启动 kubelet 来说，只需要修改 kubelet.kubeconfig、kube-proxy.kubeconfig 中的 apiserver 地址为当前 master ip 6443 端口即可

```
# grep server kube-proxy.kubeconfig
server: https://192.168.2.219:6443
# grep server bootstrap.kubeconfig
server: https://192.168.2.219:6443

systemctl daemon-reload
systemctl start kubelet
systemctl start kube-proxy
systemctl enable kubelet
systemctl enable kube-proxy
```

最终成功启动后:

![](https://raw.githubusercontent.com/jevic/images/master/kubernetes/kubernetes-get-node.png)


## 五. Calico
### 5.1 修改calico 配置

```
#wget https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/calico.yaml -O calico.example.yaml

ETCD_CERT=`cat /etc/etcd/ssl/etcd.pem | base64 | tr -d '\n'`
ETCD_KEY=`cat /etc/etcd/ssl/etcd-key.pem | base64 | tr -d '\n'`
ETCD_CA=`cat /etc/etcd/ssl/etcd-root-ca.pem | base64 | tr -d '\n'`
ETCD_ENDPOINTS="https://192.168.2.219:2379,https://192.168.2.220:2379,https://192.168.2.221:2379"

cp calico.example.yaml calico.yaml

sed -i "s@.*etcd_endpoints:.*@\ \ etcd_endpoints:\ \"${ETCD_ENDPOINTS}\"@gi" calico.yaml

sed -i "s@.*etcd-cert:.*@\ \ etcd-cert:\ ${ETCD_CERT}@gi" calico.yaml
sed -i "s@.*etcd-key:.*@\ \ etcd-key:\ ${ETCD_KEY}@gi" calico.yaml
sed -i "s@.*etcd-ca:.*@\ \ etcd-ca:\ ${ETCD_CA}@gi" calico.yaml

sed -i 's@.*etcd_ca:.*@\ \ etcd_ca:\ "/calico-secrets/etcd-ca"@gi' calico.yaml
sed -i 's@.*etcd_cert:.*@\ \ etcd_cert:\ "/calico-secrets/etcd-cert"@gi' calico.yaml
sed -i 's@.*etcd_key:.*@\ \ etcd_key:\ "/calico-secrets/etcd-key"@gi' calico.yaml

# 注释掉 calico-node 部分(由 Systemd 接管)
sed -i '123,219s@.*@#&@gi' calico.yaml

```


### 5.2 创建systemd 文件
> 需要在每个节点(master和node)执行

```
K8S_MASTER_IP="192.168.2.219"
HOSTNAME=`cat /etc/hostname`
ETCD_ENDPOINTS="https://192.168.2.219:2379,https://192.168.2.220:2379,https://192.168.2.221:2379"

cat > /lib/systemd/system/calico-node.service <<EOF
[Unit]
Description=calico node
After=docker.service
Requires=docker.service

[Service]
User=root
Environment=ETCD_ENDPOINTS=${ETCD_ENDPOINTS}
PermissionsStartOnly=true
ExecStart=/usr/bin/docker run   --net=host --privileged --name=calico-node \\
                                -e ETCD_ENDPOINTS=${ETCD_ENDPOINTS} \\
                                -e ETCD_CA_CERT_FILE=/etc/etcd/ssl/etcd-root-ca.pem \\
                                -e ETCD_CERT_FILE=/etc/etcd/ssl/etcd.pem \\
                                -e ETCD_KEY_FILE=/etc/etcd/ssl/etcd-key.pem \\
                                -e NODENAME=${HOSTNAME} \\
                                -e IP= \\
                                -e IP_AUTODETECTION_METHOD=can-reach=${K8S_MASTER_IP} \\
                                -e AS=64512 \\
                                -e CLUSTER_TYPE=k8s,bgp \\
                                -e CALICO_IPV4POOL_CIDR=10.20.0.0/16 \\
                                -e CALICO_IPV4POOL_IPIP=always \\
                                -e CALICO_LIBNETWORK_ENABLED=true \\
                                -e CALICO_NETWORKING_BACKEND=bird \\
                                -e CALICO_DISABLE_FILE_LOGGING=true \\
                                -e FELIX_IPV6SUPPORT=false \\
                                -e FELIX_DEFAULTENDPOINTTOHOSTACTION=ACCEPT \\
                                -e FELIX_LOGSEVERITYSCREEN=info \\
                                -e FELIX_IPINIPMTU=1440 \\
                                -e FELIX_HEALTHENABLED=true \\
                                -e CALICO_K8S_NODE_REF=${HOSTNAME} \\
                                -v /etc/calico/ssl:/etc/etcd/ssl \\
                                -v /lib/modules:/lib/modules \\
                                -v /var/lib/calico:/var/lib/calico \\
                                -v /var/run/calico:/var/run/calico \\
                                k8s.yfcloud.com/calico/node:v3.1.0
ExecStop=/usr/bin/docker rm -f calico-node
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```


对于以上脚本中的 K8S_MASTER_IP 变量，只需要填写一个 master ip 即可，这个变量用于 calico 自动选择 IP 使用；在宿主机有多张网卡的情况下，calcio node 会自动获取一个 IP，获取原则就是尝试是否能够联通这个 master ip

由于 calico 需要使用 etcd 存储数据，所以需要复制 etcd 证书到相关目录，/etc/calico 需要在每个节点都有

```
cp -r /etc/etcd/ssl /etc/calico/
```


### 5.3 修改 kubelet 配置
>使用 Calico 后需要修改 kubelet 配置增加 CNI 设置(--network-plugin=cni)，修改后配置如下


```
###
# kubernetes kubelet (minion) config
# The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--node-ip=192.168.2.219"
# The port for the info server to serve on
# KUBELET_PORT="--port=10250"
# You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname-override=master219"
# location of the api-server
# KUBELET_API_SERVER=""
# Add your own!
KUBELET_ARGS="  --bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \
                --cert-dir=/etc/kubernetes/ssl \
                --cgroup-driver=cgroupfs \
                --cluster-dns=10.254.0.2 \
                --network-plugin=cni \
                --cluster-domain=cluster.local. \
                --fail-swap-on=false \
                --feature-gates=RotateKubeletClientCertificate=true,RotateKubeletServerCertificate=true \
                --node-labels=node-role.kubernetes.io/master=true \
                --image-gc-high-threshold=70 \
                --image-gc-low-threshold=50 \
                --kube-reserved=cpu=500m,memory=512Mi,ephemeral-storage=1Gi \
                --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
                --system-reserved=cpu=1000m,memory=1024Mi,ephemeral-storage=1Gi \
                --serialize-image-pulls=false \
                --sync-frequency=30s \
                --pod-infra-container-image=k8s.gcr.io/pause-amd64:3.0 \
                --resolv-conf=/etc/resolv.conf \
                --rotate-certificates"
```

### 5.4 创建 Calico Daemonset

```
# 先创建 RBAC
kubectl apply -f \
https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/rbac.yaml

# 再创建 Calico Daemonset
kubectl create -f calico.yaml

```


### 5.5 启动 Calico Node

```
systemctl daemon-reload
systemctl restart calico-node
systemctl enable calico-node

# 等待 20s 拉取镜像, 可以提前将镜像拉取
sleep 20
systemctl restart kubelet
```

### 5.6 测试网络

```
# 创建 deployment
cat << EOF >> demo.deploy.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-deployment
spec:
  replicas: 5
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: demo
        image: jevic/demo
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
EOF
kubectl create -f demo.deploy.yml
```

测试结果

```
[root@master219 ~]# kubectl get pod -o wide
NAME                             READY     STATUS    RESTARTS   AGE       IP             NODE
demo-deployment-566fd7b7-fjg7p   1/1       Running   0          3d        10.20.211.67   node220
demo-deployment-566fd7b7-qz5q2   1/1       Running   0          3d        10.20.237.5    master219
demo-deployment-566fd7b7-zzqf9   1/1       Running   0          3d        10.20.206.3    node221
[root@master219 ~]# kubectl exec -it demo-deployment-566fd7b7-fjg7p ping 10.20.206.3
PING 10.20.206.3 (10.20.206.3): 56 data bytes
64 bytes from 10.20.206.3: seq=0 ttl=62 time=0.431 ms
64 bytes from 10.20.206.3: seq=1 ttl=62 time=0.257 ms
64 bytes from 10.20.206.3: seq=2 ttl=62 time=0.246 ms
64 bytes from 10.20.206.3: seq=3 ttl=62 time=0.337 ms
^C
--- 10.20.206.3 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.246/0.317/0.431 ms
```

## 六. 部署DNS
### 6.1 修改配置文件
将下载的 kubernetes-server-linux-amd64.tar.gz 解压后，再解压其中的 kubernetes-src.tar.gz 文件。
coredns 对应的目录是：`cluster/addons/dns`

```
# diff coredns.yaml coredns.yaml.base
61c61
<         kubernetes cluster.local. in-addr.arpa ip6.arpa {
---
>         kubernetes __PILLAR__DNS__DOMAIN__ in-addr.arpa ip6.arpa {
103c103
<         image: k8s.yfcloud.com/k8s.gcr.io/coredns:1.0.6
---
>         image: k8s.gcr.io/coredns:1.0.6
153c153
<   clusterIP: 10.254.0.2
---
>   clusterIP: __PILLAR__DNS__SERVER__
```

### 6.2 检查coredns

```
# kubectl get pod -n kube-system  |grep dns
coredns-787558c684-6tlwj                   1/1       Running   0          3d
coredns-787558c684-t4h6j                   1/1       Running   0          3d

# kubectl exec -it demo-deployment-566fd7b7-qz5q2 bash
bash-4.4# nslookup kubernetes
nslookup: can't resolve '(null)': Name does not resolve

Name:      kubernetes
Address 1: 10.254.0.1 kubernetes.default.svc.cluster.local

```

### 6.3 DNS 自动扩容

[下载源码文件](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.10.md#downloads-for-v11010)

- 文件路径: kubernetes/kubernetes-src1.10.1/cluster/addons/dns-horizontal-autoscaler


```
kubectl create -f dns-horizontal-autoscaler.yaml
```
## 七. Dashboard

```
wget https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml -O kubernetes-dashboard.yaml
```

### 7.1 配置端口

便于访问配置为: NodePort, 最后的修改部分如下 

```
# ------------------- Dashboard Service ------------------- #

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  type: NodePort
  ports:
    - name: dashboard-tls
      port: 443
      targetPort: 8443
      nodePort: 30000
      protocol: TCP
  selector:
    k8s-app: kubernetes-dashboard
```

执行 kubectl create -f kubernetes-dashboard.yaml 创建即可

### 7.2 创建Admin 账户

默认情况下部署成功后可以直接访问 https://NODE_IP:30000 访问，但是想要登录进去查看的话需要使用 kubeconfig 或者 access token 的方式；实际上这个就是 RBAC 授权控制，以下提供一个创建 admin access token 的脚本，更细节的权限控制比如只读用户可以参考[官方文档](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#rolebinding-and-clusterrolebinding)

```
#!/bin/bash

if kubectl get sa dashboard-admin -n kube-system &> /dev/null;then
    echo -e "\033[33mWARNING: ServiceAccount dashboard-admin exist!\033[0m"
else
    kubectl create sa dashboard-admin -n kube-system
    kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
fi

kubectl describe secret -n kube-system $(kubectl get secrets -n kube-system | grep dashboard-admin | cut -f1 -d ' ') | grep -E '^token'
```

成功后访问如下(如果访问不了的话请检查下 iptable FORWARD 默认规则是否为 DROP，如果是将其改为 ACCEPT 即可)

![](https://raw.githubusercontent.com/jevic/images/master/kubernetes/kubernetes-dashboard-sa.png)
![](https://raw.githubusercontent.com/jevic/images/master/kubernetes/kubernetes-dashboard.png)

## 八. 部署 heapster
[官方说明](https://github.com/kubernetes/heapster/blob/master/docs/influxdb.md)

- git clone https://github.com/kubernetes/heapster.git

### 8.1 grafana NodePort
在[官方部署说明](https://github.com/kubernetes/heapster/blob/master/docs/influxdb.md)中已经有提示,需要将grafana改为NodePort 进而方可使用各节点IP:Port访问，下面列出修改后的配置

```
.... 省略上面部分 .....
apiVersion: v1
kind: Service
metadata:
  labels:
    # For use as a Cluster add-on (https://github.com/kubernetes/kubernetes/tree/master/cluster/addons)
    # If you are NOT using this as an addon, you should comment out this line.
    kubernetes.io/cluster-service: 'true'
    kubernetes.io/name: monitoring-grafana
  name: monitoring-grafana
  namespace: kube-system
spec:
  # In a production setup, we recommend accessing Grafana through an external Loadbalancer
  # or through a public IP.
  # type: LoadBalancer
  # You could also use NodePort to expose the service at a randomly-generated port
  type: NodePort
  ports:
  - name: grafana-web  ## 自定义
    port: 80
    targetPort: 3000
    nodePort: 30001 ## 30000-50000 自定义
    protocol: TCP
  selector:
    k8s-app: grafana
```

### 8.2 执行部署

```
kubectl create -f deploy/kube-config/influxdb/
kubectl create -f deploy/kube-config/rbac/heapster-rbac.yaml
```

### 8.3 查看grafana
- 面板需要自己手动添加

![](https://raw.githubusercontent.com/jevic/images/master/kubernetes/kubernetes-grafana.png)
### 8.4 访问Dashboard
- 可以看到界面上面已经显示内存 CPU资源图表

![](https://raw.githubusercontent.com/jevic/images/master/kubernetes/kubernetes-dashboard-view01.png)


## 九. 其他说明:
### 9.1 命令行自动补全

```
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

### 9.2 coredns 

如果配置失败或者报错请检查 /etc/resolv.conf 域名解析配置
另外请确保已经执行了前面初始化关于内核优化的执行部分!!! 

```
cat /etc/resolv.conf
# Generated by NetworkManager
#search lan
#nameserver 192.168.1.1
nameserver 223.5.5.5
nameserver 8.8.8.8
```

### 9.3 GCR 镜像被墙
默认kubernetes 镜像被托管在Google仓库无法被正常获取,使用开源社区镜像仓库Pull 即可
建议再即将部署前将所有用到的镜像提前pull 下来.
参考[GCR Google Container Registry 镜像](https://www.jevic.cn/2018/05/25/mirror/)


### 9.4 dashboard
直接使用kubernetes-src 里面的yaml文件创建即可,记得修改NodePort

### 9.5 heapster
同样直接使用github 官方提供的说明配置即可.无需再百度或者Google其他配置范例.

### 9.6 善用帮助命令

kubernetes 有着简单易用非常清晰明了的帮助提示命令,这点是绝对备受一致好评的;

--help 不会哪里就help 哪里

```
kubectl expose --help
```

不会写yaml ? 没有关系使用下面这个命令即可获取资源定义帮助
使用'.' 分割逐级查看各对象的使用方法

```
## 例如这里获取pod 关于spec 的资源定义帮助信息
kubectl explain pod.spec
## 获取pod资源中containers的定义
kubectl explain pod.spec.containers
```

>提示: 在帮助信息的最下方也会显示出此命令或资源定义的官网详细文档链接地址,查看官方文档可更快的帮助你理解和学习!!


## 十. 参考链接
- [set-up-kubernetes-1.10.1-cluster-by-hyperkube](https://mritd.me/2018/04/19/set-up-kubernetes-1.10.1-cluster-by-hyperkube/)
- [https://k8s-install.opsnull.com](https://k8s-install.opsnull.com/)
- [https://kubernetes.feisky.xyz](https://kubernetes.feisky.xyz/)
