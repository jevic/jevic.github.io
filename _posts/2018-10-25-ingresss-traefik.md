---
layout: post
categories: Kubernetes
title: kubernetes nginx-ingress and traefik
date: 2018-10-25 22:35:46 +0800
description: Docker,kubernetes,k8s
keywords: Docker,k8s
---



在Kubernetes中，服务和Pod的IP地址仅可以在集群网络内部使用，对于集群外的应用是不可见的。为了使外部的应用能够访问集群内的服务，在Kubernetes中可以通过NodePort和LoadBalancer这两种类型的服务，或者使用Ingress。Ingress本质是通过http代理服务器将外部的http请求转发到集群内部的后端服务。

## 一. 证书(可选配置)
- 使用已购买的证书
- 配置自签名证书

```
# openssl genrsa -out jevic_key.pem 4096
Generating RSA private key, 4096 bit long modulus
.....................++
.....++
e is 65537 (0x10001)
[root@master219 ssl]# openssl req -new -x509 -key jevic_key.pem -out root.crt -days 3650
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:GD
State or Province Name (full name) []:GD
Locality Name (eg, city) [Default City]:SZ
Organization Name (eg, company) [Default Company Ltd]:jevic
Organizational Unit Name (eg, section) []:jevic
Common Name (eg, your name or your server's hostname) []:jevic.cn
Email Address []:admin@jevic.cn
[root@master219 ssl]# ls
jevic_key.pem  root.crt

```

## 二. nginx-ingress 
### 2.1 部署
如果kubernetes 集群启用了RBAC，则一定要加rbac.create=true参数
helm 安装请参考[Kubernetes helm 包管理工具](https://www.jevic.cn/2018/10/13/helm/)

```
helm install stable/nginx-ingress --set controller.hostNetwork=true，rbac.create=true
```

查看状态:

```
# helm list
NAME            	REVISION	UPDATED                 	STATUS  	CHART              	APP VERSION	NAMESPACE
quiet-abalone   	1       	Thu Nov 29 19:55:28 2018	DEPLOYED	nginx-ingress-0.9.5	0.10.2     	default

# kubectl get all --all-namespaces -l app=nginx-ingress
NAMESPACE   NAME                                                               READY     STATUS    RESTARTS   AGE
default     pod/quiet-abalone-nginx-ingress-controller-7b8b745f96-gczjl        1/1       Running   1          5d
default     pod/quiet-abalone-nginx-ingress-default-backend-54f4fb569c-ndh4x   1/1       Running   1          5d

NAMESPACE   NAME                                                  TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
default     service/quiet-abalone-nginx-ingress-controller        LoadBalancer   10.254.235.53   <pending>     80:45608/TCP,443:40984/TCP   5d
default     service/quiet-abalone-nginx-ingress-default-backend   ClusterIP      10.254.20.221   <none>        80/TCP                       5d

NAMESPACE   NAME                                                          DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
default     deployment.apps/quiet-abalone-nginx-ingress-controller        1         1         1            1           5d
default     deployment.apps/quiet-abalone-nginx-ingress-default-backend   1         1         1            1           5d

NAMESPACE   NAME                                                                     DESIRED   CURRENT   READY     AGE
default     replicaset.apps/quiet-abalone-nginx-ingress-controller-7b8b745f96        1         1         1         5d
default     replicaset.apps/quiet-abalone-nginx-ingress-default-backend-54f4fb569c   1         1         1         5d
```

### 2.2 nginx 负载均衡测试

#### 2.2.1 nginx-demo.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-demo
  labels:
    app: my-nginx
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
    name: http
  selector:
    app: my-nginx
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: my-nginx
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: my-nginx
    spec:
      containers:
      - image: nginx/nginx:1.7.9
        name: nginx-demo
        ports:
        - containerPort: 80
```

#### 2.2.2 ingress-nginx-demo.yaml

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress-demo
  namespace: default
#  annotations:
#    kubernetes.io/ingress.class: "public"
#    ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: test.nginx.com
    http:
      paths:
      - backend:
          serviceName: nginx-demo
          servicePort: 80
        path: /
```

## 三. traefik
- [官网 yaml文档](https://github.com/containous/traefik/tree/master/examples/k8s)
- [docs.traefik.io](https://docs.traefik.io/user-guide/kubernetes/)

### 3.1 traefik-insecure.yaml
- 只开启了http 请求访问

```
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: traefik-conf
  namespace: kube-system
data:
  traefik.toml: |
    insecureSkipVerify = true
    defaultEntryPoints = ["http"]
    [entryPoints]
      [entryPoints.http]
      address = ":80"
---
kind: DaemonSet
apiVersion: extensions/v1beta1
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      terminationGracePeriodSeconds: 60
      hostNetwork: true
      volumes:
      - name: config
        configMap:
          name: traefik-conf
      containers:
      - image: k8s.jevic.cn/kubernetes/traefik:1.7.3
        name: traefik-ingress-lb
        ports:
        - name: http
          containerPort: 80
          hostPort: 80
        - name: admin
          containerPort: 8080
        securityContext:
          privileged: true
        args:
        - --configfile=/config/traefik.toml
        - -d
        - --web
        - --kubernetes
        - --web.address=0.0.0.0:8081
        volumeMounts:
        - mountPath: "/config"
          name: "config"
---
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
    - protocol: TCP
      port: 80
      name: web
    - protocol: TCP
      port: 8080
      name: admin
    - protocol: TCP
      port: 443
      name: https
  type: NodePort
apiVersion: v1
kind: Service
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: ingress.jevic.cn
    http:
      paths:
      - backend:
          serviceName: traefik-web-ui
          servicePort: 80
```


```
kubectl create -f traefik-insecure.yaml
```

然后等待服务正常启动即可

```
# kubectl get pod -n kube-system -l k8s-app=traefik-ingress-lb
NAME                               READY     STATUS    RESTARTS   AGE
traefik-ingress-controller-mv8cg   1/1       Running   0          4d
traefik-ingress-controller-vvfc7   1/1       Running   0          4d
```


### 3.2 traefik-https.yaml
- 开启http 和 https 访问

#### 3.2.1 创建 secret 
- 这里使用购买的证书,也可以使用自签名证书测试；

```
kubectl create secret tls ssl --cert=jevic.cert --key=jevic.key -n kube-system
kubectl get secret -n kube-system
```
这里注意namespace 一定要配置为kube-system

#### 3.2.2 traefik-https.yaml

```
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: traefik-conf
  namespace: kube-system
data:
  traefik.toml: |
    insecureSkipVerify = true
    defaultEntryPoints = ["http","https"]
    [entryPoints]
      [entryPoints.http]
      address = ":80"
      [entryPoints.https]
      address = ":443"
        [entryPoints.https.tls]
          [[entryPoints.https.tls.certificates]]
          CertFile = "/ssl/tls.crt"
          KeyFile = "/ssl/tls.key"
---
kind: DaemonSet
apiVersion: extensions/v1beta1
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      terminationGracePeriodSeconds: 60
      hostNetwork: true
      volumes:
      - name: ssl
        secret:
          secretName: ssl
      - name: config
        configMap:
          name: traefik-conf
      containers:
      - image: k8s.jevic.cn/kubernetes/traefik:latest
        name: traefik-ingress-lb
        ports:
        - name: http
          containerPort: 80
          hostPort: 80
        - name: admin
          containerPort: 8081
        securityContext:
          privileged: true
        args:
        - --configfile=/config/traefik.toml
        - -d
        - --web
        - --kubernetes
        - --web.address=0.0.0.0:8081
        volumeMounts:
        - mountPath: "/ssl"
          name: "ssl"
        - mountPath: "/config"
          name: "config"
---
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
    - protocol: TCP
      port: 80
      name: web
    - protocol: TCP
      port: 8081
      name: admin
    - protocol: TCP
      port: 443
      name: https
  type: NodePort
apiVersion: v1
kind: Service
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
  - port: 80
    targetPort: 8081
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: ingress.jevic.cn
    http:
      paths:
      - backend:
          serviceName: traefik-web-ui
          servicePort: 80
```

默认端口: 8080 这里手动指定了web-ui端口为:8081 

```
- --web.address=0.0.0.0:8081
```

### 3.3 目录结构

```
├── jevic.cert
├── jevic.key
├── nginx-demo
│   ├── ingress-nginx-traefik.yaml
│   └── nginx-demo.yaml
├── traefik-https.yaml
└── traefik-insecure.yaml
```

### 3.4 traefik nginx-demo

#### nginx-demo.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-demo
  labels:
    app: nginx-demo
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    name: http
  - port: 443
    protocol: TCP
    targetPort: 443
    name: https
  selector:
    app: nginx-demo
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-demo
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - image: k8s.jevic.cn/nginx/nginx:1.7.9
        name: nginx-demo
        ports:
        - name: https
          containerPort: 443
        - name: http
          containerPort: 80
```

#### ingress-nginx-traefik.yaml

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress-demo
  namespace: default
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: nginx.jevic.cn
    http:
      paths:
      - backend:
          serviceName: nginx-demo
          servicePort: 80
        path: /
```

#### 部署

```
# ls
ingress-nginx-traefik.yaml  nginx-demo.yaml
# kubectl create -f nginx-demo.yaml
service "nginx-demo" created
replicationcontroller "nginx-demo" created
# kubectl create -f ingress-nginx-traefik.yaml
ingress.extensions "nginx-ingress-demo" created
# kubectl get pods -l app=nginx-demo
NAME               READY     STATUS    RESTARTS   AGE
nginx-demo-z6gz7   1/1       Running   0          13s
# kubectl get svc -l app=nginx-demo
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
nginx-demo   ClusterIP   10.254.97.231   <none>        80/TCP,443/TCP   22s
# kubectl get ingress
NAME                 HOSTS               ADDRESS   PORTS     AGE
nginx-ingress-demo   nginx.jevic.cn             80        21s
```

![](https://raw.githubusercontent.com/jevic/images/master/kubernetes/kubernetes-ingress-traefik01.png)
![](https://raw.githubusercontent.com/jevic/images/master/kubernetes/kubernetes-ingress-traefik02.png)

- [配置文件示例](https://github.com/jevic/kshell/tree/master/addon/traefik)
