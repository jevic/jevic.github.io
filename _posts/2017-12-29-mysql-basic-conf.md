---
layout: post
categories: Mysql
title: Mysql 操作杂录
date: 2017-01-12 17:39:28 +0800
description: Mysql 操作杂录
keywords: mysql
---

>MySQL 常用操作语句


## 密码修改
1. set password for root@localhost = password('password');
2. mysqladmin -uroot -p123456 password 123 
3. update user set password=password('123') where user='root' and host='localhost'; 
4. flush privileges; (改密码后执行刷新权限)

### 忘记密码
1. 关闭正在运行的MySQL服务。 
2. 转到mysql\bin目录。 
3. mysqld --skip-grant-tables 回车。--skip-grant-tables 的意思是启动MySQL服务的时候跳过权限表认证。 
4. 再开一个DOS窗口（因为刚才那个DOS窗口已经不能动了），转到mysql\bin目录。 
5. 输入mysql回车，如果成功，将出现MySQL提示符 >。 
6. 连接权限数据库： use mysql; 
7. 刷新: flush privileges;
6. 改密码：update user set password=password("123") where user="root";（别忘了最后加分号） 。 
7. 刷新权限（必须步骤）：flush privileges;　。 
8. 退出 quit。 
9. 注销系统，再进入，使用用户名root和刚才设置的新密码123登录


## 删除数据库：
	drop database db_name; 

## 创建表：
```
	create table tb_name(col1,col2,....);
	mysql> create table names(Name CHAR(20) NOT NULL,Age tinyint unsigned,Gender CHAR(1) NOT NULL);
```

- 查看数据库中的表：show tables from db_name;
- 查看表的结构：desc tb_name;
- 删除表： drop table tb_name;

## 从库只读设置(直接生效无需重启)

```
mysql> set global read_only=1;   开启
mysql> set global read_only=0;   关闭
mysql> show variables like '%read_only%';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| innodb_read_only | OFF   |
| read_only        | ON    |
| super_read_only  | OFF   |
| tx_read_only     | OFF   |
+------------------+-------+
4 rows in set (0.00 sec)

注意: 拥有super all权限的用户仍然可以写操作；

```

## 锁定数据库
```
1.FLUSH TABLES WITH READ LOCK
全局读锁定，所有库所有表都被锁定只读。一般都是用在数据库联机备份
这个时候数据库的写操作将被阻塞，读操作顺利进行。
解锁的语句也是unlock tables。

2.LOCK TABLES tbl_name [AS alias] {READ [LOCAL] | [LOW_PRIORITY] WRITE}
表级别的锁定，可以锁定某一个表。
例如： locktables test read; 不影响其他表的写操作。
解锁语句也是unlock tables

这两个语句在执行的时候都需要注意个特点，就是隐式提交的语句。
在退出MySQL终端的时候都会隐式的执行unlock tables。
也就是如果要让表锁定生效就必须一直保持对话

P.S.  MYSQL的read lock和wirte lock
read-lock:  允许其他并发的读请求，但阻塞写请求，即可以同时读，但不允许任何写。也叫共享锁
write-lock: 不允许其他并发的读和写请求，是排他的(exclusive)。也叫独占锁

```



## lock 删除以及慢查询

```
select concat('KILL ',id,';') from information_schema.processlist where TIME>100 and INFO like 'SELECT%';

select * from information_schema.`PROCESSLIST` where DB='ops_y' and state != 'Waiting for table level lock' order by TIME desc

select * from information_schema.`PROCESSLIST` where DB='ops_y'  order by TIME desc

这2条语句就能看到了，根本没有特别长时间的ddl和dml，就是有大量的lock处理不过来

https://yq.aliyun.com/articles/11033


show engines;
show processlist;

```

## 创建数据库

```
mysql> CREATE DATABASE IF NOT EXISTS cattle COLLATE = 'utf8_general_ci' CHARACTER SET = 'utf8';
Query OK, 1 row affected, 1 warning (0.01 sec)

mysql> GRANT ALL ON cattle.* TO 'cattle'@'%' IDENTIFIED BY 'cattle0c73aa94933274a86ee51d5f052b59a3';
Query OK, 0 rows affected, 1 warning (0.03 sec)

mysql> GRANT ALL ON cattle.* TO 'cattle'@'localhost' IDENTIFIED BY 'cattle0c73aa94933274a86ee51d5f052b59a3';
Query OK, 0 rows affected, 2 warnings (0.00 sec)

mysql> show grants for cattle;
+-------------------------------------------------------------+
| Grants for cattle@%                                   |
+-------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'cattle'@'%'                    |
| GRANT ALL PRIVILEGES ON `cattle`.* TO 'cattle'@'%' |
+-------------------------------------------------------------+

```


## mysqldump

```
命令行下具体用法如下： 

mysqldump -u用戶名 -p密码 -d 数据库名 表名 > 脚本名;

导出整个数据库结构和数据
mysqldump -h localhost -uroot -p123456 database > dump.sql

导出单个数据表结构和数据
mysqldump -h localhost -uroot -p123456  database table > dump.sql

导出整个数据库结构（不包含数据）
mysqldump -h localhost -uroot -p123456  -d database > dump.sql

导出单个数据表结构（不包含数据）
mysqldump -h localhost -uroot -p123456  -d database table > dump.sq


```



## 清空删除表数据

```
truncate table table_name;
delete from table_name where ;
```
* truncate是整体删除（速度较快)
* delete是逐条删除（速度较慢）
* truncate不写服务器log，delete写服务器log，也就是truncate效率比delete高的原因。
* truncate不激活trigger(触发器)，但是会重置Identity（标识列、自增字段），相当于自增列会被置为初始值，又重新从1开始记录，而不是接着原来的ID数。
* 而delete删除以后，Identity依旧是接着被删除的最近的那一条记录ID加1后进行记录。
如果只需删除表中的部分记录，只能使用DELETE语句配合where条件。 


## 删除数据库中所有表

```
SELECT concat('DROP TABLE IF EXISTS ', table_name, ';')
FROM information_schema.tables
WHERE table_schema = 'mydb';

```

## 容量大小查看
###  所有库容量
```
select table_schema,sum(data_length)/1024/1024 as data_length,sum(index_length)/1024/1024 as index_length,sum(data_length+index_length)/1024/1024 as sum from information_schema.tables;
```

### 各个库大小汇总
```
select table_schema, sum(data_length+index_length)/1024/1024 as total_mb,sum(data_length)/1024/1024 as data_mb, sum(index_length)/1024/1024 as index_mb,count(*) as tables, curdate() as today from information_schema.tables group by table_schema order by 2 desc;
```

### 查看单个库
注意分隔符

```
select concat(truncate(sum(data_length)/1024/1024,2),'mb') as data_size, \
concat(truncate(sum(max_data_length)/1024/1024,2),'mb') as max_data_size, \
concat(truncate(sum(data_free)/1024/1024,2),'mb') as data_free, \
concat(truncate(sum(index_length)/1024/1024,2),'mb') as index_size\
 from information_schema.tables where table_schema = 'test_crawl';

```

### 单个表状态

```
show table status from test_data where name = 'test_table' \G
```

### 单个库所有表状态
注意分隔符

```
select table_name, (data_length/1024/1024) as data_mb , (index_length/1024/1024) \
as index_mb, ((data_length+index_length)/1024/1024) as all_mb, table_rows \
from tables where table_schema = 'test_data';
```

- 批量执行
 
```
# cat ~/.my.cnf 
[client]
user = root
host = localhost
password = test


#!/bin/bash
DB_USER="test"
PASSWD="test"
HOST="192.168.1.236"
DATABASE="test"
Tmplog=dbsize.log
##
#Tables=`mysql -u${DB_USER} -p${PASSWD} -e "show tables from $DATABASE;" |egrep -v "Table|mysql"`
Tables=`mysql -e "show tables from $DATABASE;" |egrep -v "Table|mysql"`

rm -rf $Tmplog

for table in $Tables;do
    ## 指定用户名密码
    #code=`mysql -u${DB_USER} -p${PASSWD} -e "use information_schema;select concat(round(sum(data_length/1024/1024),2),'MB') as data_length_MB, concat(round(sum(index_length/1024/1024),2),'M
B') as index_length_MB  from tables where table_schema='"$DATABASE"' and table_name = '"$table"';"|egrep -v "length"`
    ## 配置了.my.cnf
    code=`mysql -e "use information_schema;select concat(round(sum(data_length/1024/1024),2),'MB') as data_length_MB, concat(round(sum(index_length/1024/1024),2),'MB') as index_length_MB  fr
om tables where table_schema='"$DATABASE"' and table_name = '"$table"';"|egrep -v "length"`

    echo -e "$table \t $code" >> $Tmplog
done

cat $Tmplog

```
