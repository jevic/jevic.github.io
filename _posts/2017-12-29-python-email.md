---
layout: post
categories: Python
title: Python Email 
date: 2017-01-23 15:10:31 +0800
description: Python Email
keywords: python email
---

>SMTP是发送邮件的协议，Python内置对SMTP的支持，可以发送纯文本邮件、HTML邮件以及带附件的邮件。
Python对SMTP支持有smtplib和email两个模块，email负责构造邮件，smtplib负责发送邮件

## 本地邮件服务
* yum install sendmail -y


``` python
[root@master python]# cat localmail.py 
#!/usr/bin/python
# -*- coding: UTF-8 -*-
import smtplib
from email.mime.text import MIMEText
from email.header import Header

sender = 'from@jevic.com'
receivers = ['test@test.com']  # 接收邮件，可设置为你的QQ邮箱或者其他邮箱

# 三个参数：第一个为文本内容，第二个 plain 设置文本格式，第三个 utf-8 设置编码
message = MIMEText('Python 邮件发送测试...', 'plain', 'utf-8')
message['From'] = Header("测试邮件", 'utf-8')
message['To'] =  Header("测试", 'utf-8')

subject = 'Python SMTP 邮件测试'
message['Subject'] = Header(subject, 'utf-8')


try:
    smtpObj = smtplib.SMTP('localhost')
    smtpObj.sendmail(sender, receivers, message.as_string())
    print "邮件发送成功"
except smtplib.SMTPException:
    print "Error: 无法发送邮件"



```

## 163邮箱[1]
``` bash
[root@master python]# cat email.py
#!/usr/bin/python
# -*- coding: utf-8 -*-
from email import encoders
from email.header import Header
from email.mime.text import MIMEText
from email.utils import parseaddr, formataddr
import smtplib

def _format_adds(s):
    name, addr = parseaddr(s)
    return formataddr (( \
           Header(name, 'utf-8').encode(), \
           addr.encode('utf-8') if isinstance(addr,unicode) else addr ))
               

from_addr = raw_input('From: ')
password = raw_input('Password: ')
to_addr = raw_input('To: ')
smtp_server = raw_input('SMTP server: ')

msg = MIMEText('Hi,Jevic....','plain','utf-8')
msg['From'] = _format_addr('Jevic 爱好者 <%s>' % from_addr)
msg['To'] = _format_addr('管理员 <%s>' % to_addr)
msg['Subject'] = Header('来自jevic...','utf-8').encode()

server = smtplib.SMTP(smtp_server, 25)
server.set_debuglevel(1)
server.login(from_addr,password)
server.sendmail(from_addr, [to_addr], msg.as_string())
server.quit()

```


## 163邮箱[2]
``` python
#!/usr/bin/env python
# coding=utf-8

import smtplib
from email.MIMEText import MIMEText
#from email.Utils import formatdate
from email.Header import Header
import sys
reload(sys)
sys.setdefaultencoding('utf-8')

def send_mail(toMail, subject, body):
    smtpHost = 'test.163.com'
    smtpPort = '25'
    
    fromMail = "test@test.com"
    username = "test"
    password = "password"
    encoding = 'utf-8'

    toMail=toMail.split(',')
    mail = MIMEText(body.encode(encoding),'plain',encoding)
    mail['Subject'] = Header(subject, encoding)
    mail['From'] = fromMail
    mail['To'] =",".join(toMail) 
    print mail['To']
#   mail['Date'] = formatdate()
    
    try::
        smtp = smtplib.SMTP(smtpHost, smtpPort, timeout=20)
        smtp.ehlo()
        smtp.login(username, password)
        print toMail
        smtp.sendmail(fromMail, toMail, mail.as_string())
        smtp.close()
    except Exception, data:
        print  Exception, ":", data
        print 'Error: unable to send email'
        return False
    return True

if __name__ == '__main__':
    print send_mail(sys.argv[1], sys.argv[2], sys.argv[3])


```
