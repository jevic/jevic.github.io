---
layout: post
categories: Linux Tools
title: 虚拟专用网络 openvpn 服务配置
date: 2017-12-29 14:33:17 +0800
description: 虚拟专用网络 openvpn 服务配置
keywords: openvpn vpn
---


## 配置安装环境

``` sh
[root@openvpn ~]# yum install -y gcc gcc-c++ pam-devel cmake openssl penssl-devel
[root@openvpn ~]# yum install -y lrzsz    (Secure 上传下载)

```

### 关闭SELINUX、清除防火墙设置

``` sh
[root@openvpn ~]# vim /etc/selinx/config
#disabled - No SELinux policy is loaded.
SELINUX=disabled
```
  
  
### 开启路由转发功能

``` sh
[root@openvpn ~]# echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
[root@openvpn ~]# sysctl -p
net.ipv4.ip_forward = 1

```

### 防火墙规则设置

``` sh
[root@openvpn ~]# systemctl disable firewalld
[root@openvpn ~]# systemctl stop firewalld
[root@openvpn ~]# yum install iptables-services -y
[root@openvpn ~]# iptables -F && iptables -t nat -F  （清除所有规则）
[root@openvpn ~]# iptables -t nat -nL && iptables -nL   
[root@openvpn ~]# iptables -t nat -A POSTROUTING -o eth0 -s 10.8.0.0/24 -j MASQUERADE
[root@openvpn ~]# iptables -A INPUT -p udp --dport 3307 -j ACCEPT     ## 3307为服务开放的端口
[root@openvpn ~]# service iptables save
[root@openvpn openvpn]# iptables -nL
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
ACCEPT     udp  --  0.0.0.0/0            0.0.0.0/0            udp dpt:3307
......
[root@openvpn openvpn]# iptables -t nat -nL
Chain PREROUTING (policy ACCEPT)
......       
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
MASQUERADE  all  --  10.8.0.0/24          0.0.0.0/0 

```

### 安装 LZO 和 openvpn
* 下载地址：http://pan.baidu.com/s/1kTxS4AF

``` sh
[root@openvpn ~]# tar -zxf lzo-2.06.tar.gz 
[root@openvpn ~]# tar -zxf openvpn-2.2.2.tar.gz 
[root@openvpn ~]#cd lzo-2.06
[root@openvpn lzo-2.06]# ./configure && make && make install
[root@openvpn ~]# cd openvpn-2.2.2
[root@openvpn openvpn-2.2.2]# ls
[root@openvpn openvpn-2.2.2]# ./configure 
[root@openvpn openvpn-2.2.2]# make && make install

```

## 配置openVPN
### 创建配置目录并编辑CA证书变量文件
``` sh
[root@openvpn ~]# mkdir /etc/openvpn
[root@openvpn ~]# cp -R openvpn-2.2.2/easy-rsa/ /etc/openvpn/
[root@openvpn ~]# cd /etc/openvpn/easy-rsa/2.0/

[root@openvpn 2.0]# vi vars 
export KEY_COUNTRY="CN" 
export KEY_PROVINCE="GD"
export KEY_CITY="SZ"
export KEY_ORG="YJ"
export KEY_EMAIL="test@test.cn"
export KEY_EMAIL=mail@host.domain
export KEY_CN=changeme
export KEY_NAME=changeme
export KEY_OU=changeme
export PKCS11_MODULE_PATH=changeme
export PKCS11_PIN=1234
 
```

###  初始化CA 证书
#### 导入环境变量
``` sh 
[root@openvpn 2.0]# source ./vars
NOTE: If you run ./clean-all, I will be doing a rm -rf on /etc/openvpn/easy-rsa/2.0/keys
[root@openvpn 2.0]# ./clean-all 
```

#### 初始化证书授权中心，创建CA证书，输出信息中已经引用了之前所设置的变量值，这里一路回车即可

``` sh
[root@openvpn 2.0]# ./build-ca （默认，一路回车即可）
Generating a 1024 bit RSA private key
.............++++++
.......................++++++
-----
Country Name (2 letter code) [CN]:
State or Province Name (full name) [GD]:
Locality Name (eg, city) [SZ]:
Organization Name (eg, company) [HF]:
Organizational Unit Name (eg, section) [changeme]:
Common Name (eg, your name or your server's hostname) [changeme]:
Name [changeme]:
Email Address [mail@host.domain]:
```


### 创建Diffie Hellman 参数
>Diffie Hellman 用于增强安全性，在OpenVPN是必须的！

``` sh
[root@openvpn ~]# ./build-dh  
```
### 创建服务器证书和密钥
>`server` 名称可以自行设定，这里默认为server

``` sh
[root@openvpn 2.0]# ./build-key-server server emailAddress          :IA5STRING:'mail@host.domain' 
(前面选项一路回车，下面两项手动输入”y"）
Certificate is to be certified until Jun 13 06:31:17 2025 GMT (3650 days)
Sign the certificate? [y/n]:y
1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated

```

### 创建客户端证书和密钥
>用户名称，这里默认创建一个client用户

``` sh
[root@openvpn 2.0]# ./build-key client emailAddress          :IA5STRING:'mail@host.domain' 
（前面选项一路回车，下面两项手动输入”y"）
Certificate is to be certified until Jun 13 06:31:43 2025 GMT (3650 days)
Sign the certificate? [y/n]:y      （这两项都是 “y"）
1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated

```

- 至此所需证书及密钥都已经生成完毕。

----


## 运行 openvpn 



#### 复制服务器证书文件到/etc/openvpn/

``` sh
[root@openvpn keys]# cp dh1024.pem ca.crt server.crt server.key /etc/openvpn/
[root@openvpn keys]# sz ca.crt client.crt client.key (下载到本地，客户端使用)
```


#### 复制服务器配置文件到/etc/openvpn下，并编辑添加配置相关参数：

``` sh
[root@openvpn keys]# cp /root/openvpn-2.2.2/sample-config-files/server.conf  /etc/openvpn/server.conf
[root@openvpn keys]# vi /etc/openvpn/server.conf
local 10.1.50.222    # VPN服务器地址
port 3307    # 端口号
proto udp  # 协议类型
dev tun
ca /etc/openvpn/ca.crt
cert /etc/openvpn/ltyserver.crt
key /etc/openvpn/ltyserver.key  # This file should be kept secret
dh /etc/openvpn/dh1024.pem
server 10.8.0.0 255.255.255.0  #vpn客户端地址段
ifconfig-pool-persist ipp.txt
push "route 10.1.50.0 255.255.0.0"
push "route 10.1.10.0 255.255.0.0"
push "redirect-gateway"   #使客户端的默认网关指向VPN,所有连接从VPN转发
push "dhcp-option DNS 10.1.254.1"
push "dhcp-option DNS 202.96.134.133"
keepalive 10 120
comp-lzo
persist-key
persist-tun
status openvpn-status.log
verb 3
#吊销证书参数 初始安装时无需设置,否则报错无法启动
;crl-verify /etc/openvpn/easy-rsa/2.0/keys/crl.pem  

```

####  启动服务

``` sh
[root@openvpn keys]# openvpn --config /etc/openvpn/server.conf      
.....................................................
Tue Jun 16 03:11:17 2015 Initialization Sequence Completed （初始化顺序完成）
服务端配置完成，启动程序，查看是否监听3307的udp端口
默认为udp，并且看到多出来一块网卡tun0，IP为10.8.0.1

添加开机自启：
[root@openvpn ~]# grep "openvpn" /etc/rc.d/rc.local 
/usr/local/sbin/openvpn --daemon --config /etc/openvpn/server.conf

```

#### 客户端配置
>下载客户端证书、密钥、客户端配置文件

``` sh
[root@openvpn ]# sz openvpn-2.2.2/sample/sample-config-files/client.conf
修改以下4项配置：
remote x.x.x.x 3307
ca ca.crt
cert client.crt
key client.key
```

---



## 自动创建用户脚本

``` sh
[root@ ~]# cat /etc/openvpn/easy-rsa/2.0/autoclientkey.sh
#!/bin/bash
## 创建用户证书文件
PWD=/etc/openvpn/easy-rsa/2.0
KONG="\\n"
cd $PWD

## 创建证书文件
. vars
(sleep 1
echo -e $KONG
sleep 1
echo -e $KONG
sleep 1
echo -e $KONG
sleep 1
echo -e $KONG
sleep 1
echo -e $KONG
sleep 1
echo -e "y"
sleep 1
echo -e "y")|./build-key $1

cd keys/
## 修改客户端配置文件
## cp openvpn-2.2.2/sample-config-files/client.conf /etc/openvpn/easy-rsa/2.0/keys/client.ovpn
sed -i "s/client.crt/$1.crt/g" client.ovpn
sed -i "s/client.key/$1.key/g" client.ovpn

## 打包文件并sz上传到本地
## 
tar -zcvf $1.tar.gz $1.crt $1.key ca.crt client.ovpn
sz $1.tar.gz

## 恢复默认客户端配置文件
sed -i "s/$1/client/g" client.ovpn
sed -i "s/$1/client/g" client.ovpn

```




## 常用操作

### 删除账号
>注销用户证书

``` sh
1. 进入 OpenVPN 安装目录
[root@openvpn ~]# cd /etc/openvpn/easy-rsa/2.0/ 
2. 执行 vars 命令 
[root@openvpn ~]# . ./var
3. 使用 revoke-full 命令，吊销客户端证书。命令格式为： 
[root@openvpn ~]# ./revoke-full client1 
这条命令执行完成之后，会在 keys 目录下面，生成一个 crl.pem 文件，这个文件中包含了吊销证书的名单。 
成功注销某个证书之后，可以打开　keys/index.txt 文件，可以看到被注销的证书前面，已标记为R． 
4. 确保服务端配置文件打开了 crl-verify 选项 
在服务端的配置文件 server.conf 中，加入这样一行： 
如果 server.conf 文件和 crl.pem 没有在同一目录下面，则 crl.pem 应该写绝对路径，例如: 
[root@openvpn ~]# cat /etc/openvpn/server.conf
crl-verify  /openvpn-2.0.5/easy-rsa/2.0/keys/crl.pem 

```

### 访问控制

``` sh
iptables -A FORWARD -i tun0 -s 10.10.10.0/30 -d 10.10.10.4 -j ACCEPT
```
意思是只允许10.10.10.0网段的IP访问10.10.10.4，如许访问多个IP，及添加多条命令即可。


### Openvpn的负载均衡和高可用

>搭建另一台openvpn服务器，将原来的ca.crt  ca.key server.key server.crt server.csr 复制到/etc/openvpn 目录下,在客户端配置文件当中添加3条命令：

``` sh
remote 192.168.3.96 1195     #openvpn 服务器ip
remote-random               #开启远程随机服务器
resolv-retry 60             #开启断开重连60s
``` 

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
