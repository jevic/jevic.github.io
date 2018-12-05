---
layout: post
categories: Kubernetes
title: Rancher Dashboard 管理kubernetes集群
date: 2018-11-25 20:35:46 +0800
description: Docker,kubernetes,k8s
keywords: Docker,k8s
---

>相比kubernetes Dashboard 
>Rancher管理面板简化了很多相对比较`复杂`的操作;

## 导入kubernetes 集群
 这里直接给出截图操作示例
 
点击导入集群
配置集群名称,Roles 默认即可,然后点击 Create
 
 ![](https://raw.githubusercontent.com/jevic/images/master/kubernetes/rancher-kubernetes-cluster01.png)


 ![](https://raw.githubusercontent.com/jevic/images/master/kubernetes/rancher-kubernetes-cluster02.png)

授权用户:
根据提示获取kubelet configuration file 中的用户名称

```
# cat ~/.kube/config|grep user ## /etc/kubernetes/kubelet.kubeconfig
    user: default-auth
users:
  user:
  
# kubectl create clusterrolebinding cluster-admin-binding --clusterrole cluster-admin --user default-auth
```

部署rancher agent
>需要注意的是,如果配置了SSL 认证,则最好使用第二条命令，否则可以使用第一条命令执行部署

> 此次使用第二条命令,以免出现证书不可用时忽略报错

```
curl --insecure -sfL https://192.168.2.218:8443/v3/import/wrmxhfxc7vcfznb9hssmqt9llx278ljqgfj49k5sxhtszrs68r6lmd.yaml | kubectl apply -f -
```

最后只需要等待agent 部署成功即可

 ![](https://raw.githubusercontent.com/jevic/images/master/kubernetes/rancher-kubernetes-cluster04.png)
 ![](https://raw.githubusercontent.com/jevic/images/master/kubernetes/rancher-kubernetes-cluster03.png)
