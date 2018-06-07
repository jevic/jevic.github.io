---
layout: post
categories: OPS
title: Chatops 趣味自动化BearyChat-hubot
date: 2017-02-07 12:15:44 +0800
description: chatops
keywords: chatops ops linux
---

>ChatOps 在 GitHub 上广受赞誉，是指由对话驱动的开发。将工具植入到对话当中，使用被关键插件和脚本改良过的聊天机器人，团队能够自动执行任务和协作，效果更好、成本更低、速度更快.....

>在聊天室里，团队成员输入命令来配置机器人，它们通过自定义脚本和插件来执行命令，从代码部署到安全事件响应再到团队成员提醒，范围极广。随着命令被不断执行，整个团队协作也实时进行。


- Hubot：GitHub 开发的机器人，用 CoffeeScript 和 Node.js 写成
- Lita：用 Ruby 写的机器人
- Err：用 Python 写的机器人


----


>BearyChat 是一款团队内部的 IM 沟通工具，旨在为团队提供一种崭新的工作方式，打通团队内部使用的众多第三方服务，提高沟通效率。

>Hubot 是 GitHub 开源的聊天机器人，提供了一种崭新的运维工作方式：配置，部署，报表，监控等。这些通过指令实现的交互方式，可以大大帮我们减少一些重复的劳作，提高工作效率，也使得工作方式一步步自动化，让工作者找到一种更愉悦的操作方式，当然这不能影响服务的稳定性。

## 部署机器人

### 1.安装nodejs
``` sh
yum安装：
[root@node1 ~]# yum install -y epel-release
[root@node1 ~]# yum install -y npm
[root@node1 ~]# npm -version
3.10.8
[root@node1 ~]# node -v
v6.9.1


二进制包安装最新版：
官网下载地址：https://nodejs.org/en/download/

[root@node1 ~]# tar xf node-v6.9.5-linux-x64.tar.xz
[root@node1 ~]# mv node-v6.9.5-linux-x64 /usr/local/nodejs
[root@node1 ~]# vim /etc/profile
export NODE_HOME=/usr/local/nodejs
export PATH=$NODE_HOME/bin:$PATH
[root@node1 ~]# source /etc/profile
[root@node1 ~]# npm -version
3.10.10
[root@node1 ~]# node -v
v6.9.5

```

## 2. 安装hubot
``` bash
[root@node1 ~]# npm install -g hubot coffee-script yo generator-hubot
[root@node1 ~]# mkdir myhubot && cd myhubot
[root@node1 myhubot]# yo hubot  (提示没有权限)
/usr/local/nodejs/lib/node_modules/yo/node_modules/mkdirp/index.js:90
                    throw err0;
                    ^

Error: EACCES: permission denied, mkdir '/root/.config'
    at Error (native)
    at Object.fs.mkdirSync (fs.js:922:18)
    at sync (/usr/local/nodejs/lib/node_modules/yo/node_modules/mkdirp/index.js:71:13)
    at Function.sync (/usr/local/nodejs/lib/node_modules/yo/node_modules/mkdirp/index.js:77:24)
    at Object.get (/usr/local/nodejs/lib/node_modules/yo/node_modules/configstore/index.js:38:13)
    at Object.Configstore (/usr/local/nodejs/lib/node_modules/yo/node_modules/configstore/index.js:27:44)
    at new Insight (/usr/local/nodejs/lib/node_modules/yo/node_modules/insight/lib/index.js:37:34)
    at Object.<anonymous> (/usr/local/nodejs/lib/node_modules/yo/lib/cli.js:172:11)
    at Module._compile (module.js:570:32)
    at Object.Module._extensions..js (module.js:579:10)

[root@node1 myhubot]# chmod g+w /root   (修改root目录权限)
[root@node1 myhubot]# yo hubot
? ==========================================================================
We're constantly looking for ways to make yo better! 
May we anonymously report usage statistics to improve the tool over time? 
More info: https://github.com/yeoman/insight & http://yeoman.io
========================================================================== Yes
                     _____________________________  
                    /                             \ 
   //\              |      Extracting input for    |
  ////\    _____    |   self-replication process   |
 //////\  /_____\   \                             / 
 ======= |[^_/\_]|   /----------------------------  
  |   | _|___@@__|__                                
  +===+/  ///     \_\                               
   | |_\ /// HUBOT/\\                             
   |___/\//      /  \\                            
         \      /   +---+                            
          \____/    |   |                            
           | //|    +===+                            
            \//      |xx|                            

? Owner XXX <XXX@163.com>
? Bot name myhubot
? Description A simple helpful robot for your Company
? Bot adapter campfire
   create bin/hubot
   create bin/hubot.cmd
   create Procfile
   create README.md
   create external-scripts.json
   create hubot-scripts.json
   create .gitignore
   create package.json
   create scripts/example.coffee
   create .editorconfig
                     _____________________________  
 _____              /                             \ 
 \    \             |   Self-replication process   |
 |    |    _____    |          complete...         |
 |__\\|   /_____\   \     Good luck with that.    / 
   |//+  |[^_/\_]|   /----------------------------  
  |   | _|___@@__|__                                
  +===+/  ///     \_\                               
   | |_\ /// HUBOT/\\                             
   |___/\//      /  \\                            
         \      /   +---+                            
          \____/    |   |                            
           | //|    +===+                            
            \//      |xx|                            

myhubot@0.0.0 /root/myhubot
├─┬ hubot@2.19.0 
│ ├── async@0.9.2 
│ ├─┬ chalk@1.1.3 
│ │ ├── ansi-styles@2.2.1 
│ │ ├── escape-string-regexp@1.0.5 
│ │ ├─┬ has-ansi@2.0.0 
│ │ │ └── ansi-regex@2.1.1 
│ │
│ │ ├─┬ connect@2.30.2 
......
```

``` bash
[root@node1 myhubot]# chmod g-w /root　（将root目录权限恢复原状）
[root@node1 myhubot]# npm install hubot-bearychat --save
myhubot@0.0.0 /root/myhubot
└─┬ hubot-bearychat@0.3.1 
  ├─┬ bearychat@0.1.0 
  │ ├── babel-plugin-add-module-exports@0.2.1 
  │ ├─┬ isomorphic-fetch@2.2.1 
  │ │ ├─┬ node-fetch@1.6.3 
  │ │ │ ├─┬ encoding@0.1.12 
  │ │ │ │ └── iconv-lite@0.4.15 
  │ │ │ └── is-stream@1.1.0 
  │ │ └── whatwg-fetch@2.0.2 
  │ └── urijs@1.18.5 
  └─┬ ws@1.1.1 
    ├── options@0.0.6 
    └── ultron@1.0.2
```

### 安装shell 命令脚本支持模块：
``` bash
[root@node1 myhubot]# npm install hubot-script-shellcmd
myhubot@0.0.0 /root/myhubot
└── hubot-script-shellcmd@0.0.16
[root@node1 myhubot]# cp -R node_modules/hubot-script-shellcmd/bash ./

修改一下external-scripts.json，添加模块：hubot-script-shellcmd
[root@node1 myhubot]# cat external-scripts.json 
[
  "hubot-diagnostics",
  "hubot-help",
  "hubot-heroku-keepalive",
  "hubot-google-images",
  "hubot-google-translate",
  "hubot-pugme",
  "hubot-maps",
  "hubot-redis-brain",
  "hubot-rules",
  "hubot-shipit",
  "hubot-script-shellcmd"  //新增部分
]
[root@node1 myhubot]# cd bash/handlers/
[root@node1 handlers]# ls
helloworld  update

新增脚本
[root@node1 handlers]# cat disk 
#!/bin/bash
df -h
[root@node1 handlers]# chmod +x disk 

```

## 3. 注册bearychat添加机器人
* https://www.bearychat.com  注册并登陆
![这里写图片描述](http://img.blog.csdn.net/20170207143342629?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![这里写图片描述](http://img.blog.csdn.net/20170207143420864?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![这里写图片描述](http://img.blog.csdn.net/20170207143434879?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 注意：
* 复制Hubot Token 这个很重要
* Hubot URL 端口可以随意指定【export EXPRESS_PORT=9090】

### 下载bearychat客户端或者直接使用网页版都可以

## 配置hubot并启动
``` bash
此处的token就是第三步添加机器人时显示的token多个使用逗号分隔
[root@node1 myhubot]# export HUBOT_BEARYCHAT_TOKENS=bd00e55956a3759886c21c1ac1fd17dd
[root@node1 myhubot]# export HUBOT_BEARYCHAT_MODE=rtm
[root@node1 myhubot]# export HUBOT_SHELLCMD_KEYWORD=run //命令别名
[root@node1 myhubot]# export EXPRESS_PORT=9090
[root@node1 myhubot]# nohup ./bin/hubot -a bearychat 2&1> hubot.log
[root@node1 myhubot]# netstat -tnlp|grep 9090
tcp        0      0 0.0.0.0:9090            0.0.0.0:*               LISTEN      54886/node 

为了以后启动方便可以直接将变量写人文件：
[root@node1 myhubot]# cat /etc/profile.d/myhubot.sh 
export HUBOT_BEARYCHAT_TOKENS=bd00e5595121249886c21c1ac1fd17dd
export HUBOT_BEARYCHAT_MODE=rtm
export EXPRESS_PORT=9090
[root@node1 myhubot]# source /etc/profile 
这样以后就无需手动再次export

```

## 5. 客户端登录并测试
![这里写图片描述](http://img.blog.csdn.net/20170207144158897?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 6. 机器人脚本示例
``` bash
[root@node1 ~]# cd myhubot/scripts
[root@node1 scripts]# vim example.coffee   (编辑并修改) 
....
#   Uncomment the ones you want to try and experiment with.
#
#   These are from the scripting documentation: https://github.com/github/hubot/blob/master/docs/scripting.md

module.exports = (robot) ->

   robot.hear /jevic/i, (res) ->
     res.send "https://jevic.github.io"
 
#   robot.respond /open the (.*) doors/i, (res) ->
....
```

![这里写图片描述](http://img.blog.csdn.net/20170207161248378?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

* 重启hubot

``` sh
[root@node1 ~]# nohup ./bin/hubot -a bearychat 2&1>> hubot.log &
```

### 测试
![这里写图片描述](http://img.blog.csdn.net/20170207161351456?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhmXzY2ODg5OQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)




* [用slack和hubot搭建你自己的运维机器人](https://segmentfault.com/a/1190000006681056#articleHeader1)
* [hubot-bearychat](https://github.com/bearyinnovative/hubot-bearychat/blob/master/README_CN.md)
* https://hubot.github.com


转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
