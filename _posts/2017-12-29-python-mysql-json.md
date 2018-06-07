---
layout: post
categories: Python
title: Python Mysql 数据格式化 Json
date: 2017-08-02 15:22:52 +0800
description: Python Mysql 数据格式化 Json
keywords: python json mysql
---

>获取MySQL的数据并转化为json格式输出


``` python
import mysql.connector
import json

User = 'admin'
Host = 'web.mysql.com'
Passwd ='Password'
db = mysql.connector.connect(user=User,host=Host,password=Passwd,database='website')
cursor = db.cursor()

cursor.execute('select * from url_article where stype="es"')
values = cursor.fetchall()

def JsonData(values):
    SerList = []
    SerData = {}

    for value in values:
        server = {}
        server['id'] = value[0]
        server['name'] = value[1]
        server['url'] = value[2]
        server['remark'] = value[3]
        server['stype'] = value[4]
        server['role'] = value[5]
        SerList.append(server)

    SerData['data'] = 0
    SerData['server'] = SerList
    jsonStr = json.dumps(SerData)
    return jsonStr

getstr = JsonData(values)
print getstr

```


