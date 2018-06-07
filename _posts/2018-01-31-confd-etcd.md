---
layout: post
categories: Bigdata
title: etcd-confd 配置管理
date: 2018-01-31 14:40:58 +0800
description: etcd-confd 配置管理
keywords: etcd confd
---

# confd
- [下载](https://github.com/kelseyhightower/confd/releases)
- 解压放到 /usr/local/bin 即可

## demo
### 创建目录

``` sh
mkdir /etc/confd/{conf.d,templates}
```


### 配置文件

``` sh

[root@master confd]# cat conf.d/jevic-cn.toml
[template]
src = "jevic-conf.tmpl"
dest = "/tmp/jevic.conf"
keys = [
    "/config/myapp/jevic/database/upstream",
    "/config/myapp/jevic/database/hosts",
    "/config/myapp/jevic/database/domain"
]
```

### 模板
![](http://ok6h8mla5.bkt.clouddn.com/etcd-confd-templates.jpg)


### 生成配置

``` sh
[root@master ~]# confd -onetime -backend etcd -node http://192.168.3.27:2379

## 间隔60s 刷新
[root@master ~]# confd -interval=60 -backend etcd -node http://192.168.3.27:2379

[root@master ~]# cat /tmp/jevic.conf
upstream jevic{
     server 127.0.0.1:5001;
}

server {
    listen 80;
    server_name www.jevic.cn;
        location / {
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_buffering off;
            proxy_pass http://jevic;
    }
}
```



转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
