---
layout: post
categories: Python
title: Python ftp 文件操作
date: 2017-06-19 15:22:52 +0800
description: Python ftp
keywords: python ftp
---

Python中默认安装的ftplib模块定义了FTP类,可用来实现简单的ftp客户端，用于上传或下载文件

[官方文档ftplib](https://docs.python.org/2/library/ftplib.html)

## FTP 连接操作
``` bash
from ftplib import FTP            #加载ftp模块
ftp=FTP()                         #设置变量
ftp.set_debuglevel(2)             #打开调试级别2，显示详细信息
ftp.connect("IP","port")          #连接的ftp sever和端口
ftp.login("user","password")      #连接的用户名，密码
print ftp.getwelcome()            #打印出欢迎信息
ftp.cmd("xxx/xxx")                #进入远程目录
bufsize=1024                      #设置的缓冲区大小
filename="filename.txt"           #需要下载的文件
file_handle=open(filename,"wb").write #以写模式在本地打开文件
ftp.retrbinaly("RETR filename.txt",file_handle,bufsize) #接收服务器上文件并写入本地文件
ftp.set_debuglevel(0)             #关闭调试模式
ftp.quit()                        #退出ftp
```

## FTP 命令操作

``` bash
ftp.cwd(pathname)                 #设置FTP当前操作的路径
ftp.dir()                         #显示目录下所有目录信息
ftp.nlst()                        #获取目录下的文件
ftp.mkd(pathname)                 #新建远程目录
ftp.pwd()                         #返回当前所在位置
ftp.rmd(dirname)                  #删除远程目录
ftp.delete(filename)              #删除远程文件
ftp.rename(fromname, toname)      #将fromname修改名称为toname。
ftp.storbinaly("STOR filename.txt",file_handel,bufsize)  #上传目标文件
ftp.retrbinary("RETR filename.txt",file_handel,bufsize)  #下载FTP文件

```

- FTP.quit()与FTP.close()的区别
    - FTP.quit():发送QUIT命令给服务器并关闭掉连接。这是一个比较“缓和”的关闭连接方式，但是如果服务器对QUIT命令返回错误时，会抛出异常。
    - FTP.close()：单方面的关闭掉连接，不应该用在已经关闭的连接之后，例如不应用在FTP.quit()之后。


## 示例代码

``` python
#!/usr/bin/env python
# coding: utf-8

import os,sys
import ftplib
from StringIO import StringIO
from ConfigParser import ConfigParser

config = ConfigParser()
config.read('config.ini')

FtpHost = config.get('FTP','HOST')
username = config.get('FTP','USER')
passwd = config.get('FTP','PASSWD')
FtpPort = config.get('FTP','PORT')

def fn(n):
    pass

class MyFtp():
    ftp = ftplib.FTP()

    def __init__(self):
        try:
            self.bufsize = 1024
            self.ftp.connect(FtpHost, FtpPort)
            self.ftp.login(username, passwd)
        except Exception, e:
            print e

    def List(self, path=''):
        def callback(str):
            print str.split(" ")[-1]

        self.ftp.dir(path, callback)

    def dirs(self, remote):
        try:
            self.ftp.cwd(remote)
        except ftplib.error_perm:
            try:
                self.ftp.mkd(remote)
                self.ftp.cwd(remote)
            except ftplib.error_perm:
                print "U have no authority to make dir"

    def Size(self, remote):
        return self.ftp.size(remote)

    def DownFile(self, remote, local):
        fp = open(local, 'wb')
        self.ftp.retrbinary('RETR ' + remote, fp.write, self.bufsize)
        fp.close()

    def Rename(self, remote):
        self.ftp.rename(remote, str.replace(remote, ".tmp", ""))

    def Upload(self, remote, local):
        def callback(str):
            return str.split(" ")[-1]

        fp = open(local, 'rb')
        self.ftp.storbinary('STOR ' + remote, fp, self.bufsize)
        return self.ftp.dir(remote, callback)
        # self.ftp.rename(remote, str.replace(remote, ".tmp", ""))
        fp.close()
```

[ftp.py](https://github.com/jevic/logcenter/blob/master/logupload/ftp.py)
