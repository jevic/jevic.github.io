---
layout: post
categories: Tools
title: shadowsocks 翻墙
date: 2018-03-19 16:41:59 +0800
description: google
keywords: google,shadowsocks
---


> 国外一台机器配置了shadowsocks,以此进行科学上网


## Server 端配置
### 一键安装脚本

```
wget --no-check-certificate -O shadowsocks-go.sh https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocks-go.sh
chmod +x shadowsocks-go.sh
./shadowsocks-go.sh 2>&1 | tee shadowsocks-go.log

```

- 配置文件
```
vim /etc/shadowsocks/config.json
{
    "server":"0.0.0.0",
    "server_port":xxxx,
    "local_port":1080,
    "password":"xxxxx",
    "method":"aes-256-cfb",
    "timeout":600
}
```

- 启动
```
/etc/init.d/shadowsocks start
```

### 卸载

```
./shadowsocks-go.sh uninstall

```

## 客户端
### Windows 客户端
- https://github.com/shadowsocks/shadowsocks-windows/releases

### Linux 客户端配置
- 安装 扩展源,pip
```
yum install epel-relase
yum install python-pip
```

- 安装shadowsocks

```
pip install shadowsocks
```

- 添加连接配置

> 默认没有配置文件需要手动添加

```
vim /etc/shadowsocks/shadowsocks.json
{
    "server":"x.x.x.x", ## 服务端IP
    "server_port":xxxx, ## 服务端口
    "local_address": "127.0.0.1", ## 0.0.0.0
    "local_port":1080,
    "password":"xxxx",
    "method":"aes-256-cfb",
    "timeout":600
}
```

- 启动文件
```
cat > /lib/systemd/system/shadowsocks.service<<EOF
[Unit]
Description=Shadowsocks
[Service]
TimeoutStartSec=0
ExecStart=/usr/bin/sslocal -c /etc/shadowsocks/shadowsocks.json
[Install]
WantedBy=multi-user.target
EOF
```

- 启动管理
```
systemctl daemon-reload
systemctl start shadowsocks
systemctl enable shadowsocks
```
- 测试连接

```
[root@master218 ~]# curl --socks5 127.0.0.1:1080 http://httpbin.org/ip
{
  "origin": "x.x.x.x"  ## 返回你的服务器地址,状态ok
}
```

### Privoxy
#### 安装
```
yum install -y privoxy
systemctl enable privoxy
systemctl start privoxy
```

#### 修改配置文件
```
[root@master218 ~]# tail -n 1 /etc/privoxy/config
forward-socks5t / 127.0.0.1:1080 .
```

- 配置环境变量

```
[root@master218 ~]# vim /etc/profile
PROXY_HOST=127.0.0.1
export all_proxy=http://$PROXY_HOST:8118 ## privoxy 默认端口8118
export ftp_proxy=http://$PROXY_HOST:8118
export http_proxy=http://$PROXY_HOST:8118
export https_proxy=http://$PROXY_HOST:8118
export no_proxy=localhost,172.16.0.0/16,192.168.0.0/16.,127.0.0.1,10.10.0.0/16

[root@master218 ~]# source /etc/profile
```

- 测试连接

```
[root@master218 ~]# curl -I www.google.com
HTTP/1.1 200 OK
......
Accept-Ranges: none
Vary: Accept-Encoding
Proxy-Connection: keep-alive

```

- 取消代理

```
while read var; do unset $var; done < <(env | grep -i proxy | awk -F= '{print $1}')
```

>参考链接:
>>https://teddysun.com/392.html


转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
