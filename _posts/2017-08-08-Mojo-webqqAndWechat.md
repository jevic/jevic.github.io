---
layout: post
categories: OPS
title: 微信 以及 WebQQ 客户端框架
date: 2017-08-08 19:56:06 +0800
description: wechat,qq,ops
keywords: alarm,wechat,qq
---


#  微信客户端
- 使用Perl语言编写的微信客户端框架，基于Mojolicious，要求Perl版本5.10+，可通过插件提供基于HTTP协议的api接口供其他语言或系统调用


- `Github项目` :     
    - [Mojo-Weixin](https://github.com/jevic/Mojo-Weixin)
    - [Mojo-qq](https://github.com/sjdy521/Mojo-Webqq)
- 详细内容参考项目首页

<!-- more -->

## 环境配置:
1. Yum

```
$ yum install -y perl-CPAN openssl-devel gcc gcc-c++
$ cpan -i App::cpanminus
$ cpanm Mojo::Weixin

```

2. [Docker](https://github.com/jevic/Mojo-Weixin/blob/master/Docker.md)
   - 建议使用Docker部署

# 启动应用

### 微信服务端
```
[root@alarm mnt]# cat server.pl
#!/usr/bin/env perl
use Mojo::Weixin;
my ($host,$port,$post_api);

$host = "0.0.0.0"; #发送消息接口监听地址，没有特殊需要请不要修改
$port = 3000;      #发送消息接口监听端口，修改为自己希望监听的端口
#$post_api = 'http://xxxx';  #接收到的消息上报接口，如果不需要接收消息上报，可以删除或注释此行

my $client = Mojo::Weixin->new(log_level=>"info",http_debug=>0);
$client->load("ShowMsg");
$client->load("Openwx",data=>{listen=>[{host=>$host,port=>$port}], post_api=>$post_api});
$client->run();


运行:
[root@alarm mnt]#  perl server.pl
[17/07/15 17:31:55] [info] 当前正在使用 Mojo-Weixin v1.3.5
[17/07/15 17:31:55] [info] 客户端加载cookie[ /tmp/mojo_weixin_cookie_default.dat ]
[17/07/15 17:31:55] [info] 执行插件[ Mojo::Weixin::Plugin::ShowMsg ]
[17/07/15 17:31:55] [info] 执行插件[ Mojo::Weixin::Plugin::Openwx ]
[17/07/15 17:31:55] [info] Listening at "http://0.0.0.0:3000"
Server available at http://0.0.0.0:3000
[17/07/15 17:31:55] [info] 客户端准备登录...
[17/07/15 17:31:56] [info] 二维码已下载到本地[ /tmp/mojo_weixin_qrcode_default.jpg ]
[17/07/15 17:31:56] [info] 等待手机微信扫描二维码...

```

### QQ服务端

```
#!/usr/bin/env perl
 use Mojo::Webqq;
 my ($host,$port,$post_api);

 $host = "0.0.0.0"; #发送消息接口监听地址，没有特殊需要请不要修改
 $port = 5000;      #发送消息接口监听端口，修改为自己希望监听的端口
 #$post_api = 'http://xxxx';  #接收到的消息上报接口，如果不需要接收消息上报，可以删除或注释此行

 my $client = Mojo::Webqq->new(log_level=>"info",tmpdir=>'/mnt/qq/tmp' );
 $client->load("ShowMsg");
 $client->load("Openqq",data=>{listen=>[{host=>$host,port=>$port}], post_api=>$post_api});
 $client->run();

```
- Note: 下载二维码扫码登陆即可


---


### 接口调用示例:
#### Shell
```
#!/bin/bash
wechat()
{
        API_ADDR='127.0.0.1:3000'
        gnum='微信报警群'
        #处理下编码，用于合并告警内容的标题和内容
        #中文需要utf8编码并进行urlencode
        message=$(echo -e "$title\n$content"|od -t x1 -A n -v -w1000000000 | tr " " %)
        API_URL="http://$API_ADDR/openwx/send_group_message?displayname=$gnum&content=$message"
        /usr/bin/curl $API_URL
}

title='来自微信报警'
content='test'
wechat


```

#### Python

```
#!/usr/bin/python
# coding: utf-8
from urllib import quote
import urllib2
import json
import sys

reload(sys)
sys.setdefaultencoding('utf-8')

def wechat(title,content):
        API_ADDR='127.0.0.1:3000'
        gnum='微信报警群'
        Title = quote(title.encode('utf8'))
        Content = quote(content.encode('utf8'))
        API_URL="http://%s/openwx/send_group_message?displayname=%s&content=%s%s" % (API_ADDR,gnum,Title,Content)
        req = urllib2.Request(API_URL)
        result = urllib2.urlopen(req)
        res = result.read()

title = 'test'
content = 'test'
wechat(title,content)

```

- Note: 详情查看API文档
    - [weixin](https://github.com/sjdy521/Mojo-Weixin/blob/master/API.md)
    - [qq](https://github.com/sjdy521/Mojo-Webqq/blob/master/API.md)

## 参考链接
- [itchat](https://itchat.readthedocs.io/zh/latest/)
- [微信机器人](http://www.jianshu.com/p/5d4de51f5375)
- [hubot-bearychat](https://github.com/bearyinnovative/hubot-bearychat/blob/master/README_CN.md)
- [chatops](https://jevic.github.io/2017/02/07/chatops/)



