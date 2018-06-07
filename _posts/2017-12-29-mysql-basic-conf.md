---
layout: post
categories: Mysql
title: Mysql 操作杂录
date: 2017-01-12 17:39:28 +0800
description: Mysql 操作杂录
keywords: mysql
---

# MySQL 操作语句
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
6. 连接权限数据库： use mysql; 。 
6. 改密码：update user set password=password("123") where user="root";（别忘了最后加分号） 。 
7. 刷新权限（必须步骤）：flush privileges;　。 
8. 退出 quit。 
9. 注销系统，再进入，使用用户名root和刚才设置的新密码123登录


### 创建数据库：
    create database db_name;
    create database if not exists db_name; 如果不存在则创建数据库
### 删除数据库：
    drop database db_name; 
### 创建表：
```
    create table tb_name(col1,col2,....);
    mysql> create table names(Name CHAR(20) NOT NULL,Age tinyint unsigned,Gender CHAR(1) NOT NULL);
```

- 查看数据库中的表：show tables from db_name;
- 查看表的结构：desc tb_name;
- 删除表： drop table tb_name;

### 从库只读设置(直接生效无需重启)

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
### 锁定数据库
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



## 查询重复记录并导出去重后的数据
```
select *,from_unixtime(`timestamp`) FROM `nls_log_merge` where host='jevic.cn' and length=5  group by `timestamp` HAVING count(*)>1 order by `timestamp` into outfile "/tmp/7.sql";
```
## 删除有重复记录的数据
```
delete FROM `nls_log_merge` WHERE host='jevic.cn' and length=5 and `timestamp` in (select a from (select `timestamp` as a FROM nls_log_merge where host='jevic.cn' and length=5  group by `timestamp` HAVING count(*)>1) a )
``` 
## 导入正确的数据
```
LOAD DATA INFILE "/tmp/hls1.sql" INTO TABLE nls_log_merge_test;
```



