---
title: MySQL部署使用
date: 2018-11-04 12:12:28
tags:
- mysql
---

## MySQL部署使用
### 1. 用户操作
- root密码设置
```bash
mysql -uroot -p
mysql> use mysql;
mysql> update user set password=PASSWORD("xinyue") where User='root';
FLUSH PRIVILEGES;
#添加远程访问
update user set host = '%' where user = 'root';

```
<!--more-->
- 查看用户
```bash
SELECT User, Host FROM mysql.user;
```
- 删除用户名为空的用户，不然本地账户无法使用密码登陆
```bash
delete from user where User='';
FLUSH PRIVILEGES;
```
- 创建用户
```bash
CREATE USER 'username'@'host' IDENTIFIED BY 'password';
```
- 删除用户
```bash
drop user username@'%'
```
- 更改用户密码
```bash
SET PASSWORD FOR name=PASSWORD('new password');
```
- 更改用户名
```bash
rename user 'jack'@'%' to 'jim'@'%';
```

### 2. 数据库操作
- 查看数据库
```bash
show databases;
```
- 选择数据库
```bash
use dong;
```
- 新建数据库
```bash
create database lrdong;
```
- 删除数据库
```bash
drop database lrdong;
```
- 查看数据库中的表
```bash
show tables;
```
- 为用户分配数据库
```bash
grant all privileges on lrdong.* to 'lr'@'%';
FLUSH PRIVILEGES;

```

### 3. 创建用户和数据库
```bash
create database showddgo;

CREATE USER 'showddgo'@'%' IDENTIFIED BY 'lixuliang186';

grant all privileges on weq.* to 'wei'@'%';

FLUSH PRIVILEGES;
```
### 4. 数据库备份脚本
- 创建备份用户
```bash
mysql> CREATE USER back_user IDENTIFIED BY 'uH6nx9p';
mysql> GRANT ALL ON *.* TO 'back_user'@'127.0.0.1' IDENTIFIED BY 'uH6nx9p';
mysql> FLUSH PRIVILEGES;
```
- 执行脚本

```bash
#!/bin/bash
#backup mysql
. /etc/profile

backuser='back_user'
backpass='uH6nx9p'
backdir='/var/local/data'
mysql='/usr/bin/'


now=`date +%Y%m%d%H%M%S`
[ ! -d ${backdir} ] && mkdir ${backdir}

find ${backdir} -type f -mtime +15 -name "*.sql" -exec rm {} \;

dblist=`${mysql}/mysql -u${backuser} -h127.0.0.1 -p${backpass} -e 'show databases;' |egrep -o '\w+' |sed -e '1d' -e '/information_schema/d' -e '/performance_schema/d'`


${mysql}/mysqldump -u${backuser} -h127.0.0.1 -p${backpass} --opt --routines --single-transaction -A > ${backdir}/all_${now}.sql

for db in $dblist;do
 ${mysql}/mysqldump -u${backuser} -h127.0.0.1 -p${backpass} --opt --routines --single-transaction -B ${db} > ${backdir}/${db}_${now}.sql
  #s3cmd put ${backdir}/${db}_${now}.sql s3://backup/
  echo "${backdir}/${db}_${now}.sql" >> /root/.backdblog
  #rm -f ${backdir}/${db}_${now}.sql
done

```

- 定时任务
```bash
echo "00 02 * * * /usr/bin/sh /root/.shell/back_mysql.sh > /dev/null 2>&1" >>  /var/spool/cron/root
service crond restart

```

##  MySQL主从配置

### 1. 主从复制的类型

- 基于语句的复制
主数据库上的执行的语句在从服务器上再执行一边
- 基于行的复制
把主服务器上面改变后的内容直接复制过去
- 基于binlog日志的方式主从复制
#### 1.1 复制原理

- master把数据改变记录到二进制日志
- slave通过I/O线程读取master中的二进制日志，并且写入到自己的中继`rlay log`日志
- slave重做中继日志中的事件，进行一条条执行，完成数据同步，数据重放

![enter image description here](http://static.roncoo.com/face/A53MzTmwcmA6ZZweaj8waTjW8kyAbmj3.jpg)

#### 1.2 注意问题

- 主从服务器操作系统环境要一致
- 主从数据版本要一致
- 主从数据库中数据一致
- 主从server id要唯一
- 主从数据库中从库配置为只读`read_only= 1`

### 2. MySQL安装

- 添加yum源
```bash
wget https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
yum install -y https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm


```
- 检查激活的订阅列表
```bash
yum repolist enabled | grep "mysql.*-community.*"
```
- 如何安装特定版本
某一个特定版本MySQL对应一个yum源
```bash
#查看列表
yum repolist all | grep mysql
#切换版本5.7切换为5.6
yum -y install yum-utils
yum-config-manager --disable mysql57-community
yum-config-manager --enable mysql56-community
#进行验证切换
yum repolist enabled | grep mysql
```
- 安装MySQL
```bash
yum install mysql-community-server -y
```
- 启动MySQL

```bash
systemctl start mysqld
systemctl enable mysqld

```
- 获取临时root密码并更改

```bash
#获取
grep 'temporary password' /var/log/mysqld.log
#登录后更改密码
mysql>  ALTER  USER  'root'@'localhost'  IDENTIFIED  BY  'Xxiny!ue12';
#关闭密码格式验证
uninstall plugin validate_password;
#接下来可以设置简单密码
 ALTER  USER  'root'@'localhost'  IDENTIFIED  BY  '123456';
#开启远程登陆
use mysql;
update user set host = '%' where user = 'root';
FLUSH PRIVILEGES;

```

- 重置密码mysql5.7

```bash
vim /etc/my.cnf
#在[mysqld]下面
skip-grant-tables
#重启
service mysqld restart
#空密码登陆
mysql -uroot -p
#选库
use mysql;
#改密码
update user set authentication_string=password('123456') where user='root';
flush privileges;
quit;
#取消skip-grant-tables
SELECT User, Host FROM mysql.user;

```

- mysql5.5

设置一个root密码

```bash
mysqladmin -u root password 'p40xdP23H0NX54'
use mysql;
SELECT User, Host FROM mysql.user;
flush privileges;
```

#### 数据存储位置

```bash
systemctl stop mysqld
mkdir /data/mysql-data/
mv /var/lib/mysql/* /data/mysql-data/
chown mysql:mysql -R /data/mysql-data
vim /etc/my.cnf
datadir=/data/mysql-data

```

### 3. 主从配置

#### 3.1 主服务器配置
- /etc/my.cnf配置
```bash
#设定server id
server_id=101
#开启二进制日志
log_bin=/var/log/mysql/mysql-bin
#单独同步数据库
binlog_do_db=db1
binlog_do_db=db2
binlog_do_db=db3
```
- 创建二进制log目录
```bash
mkdir /var/log/mysql
chown mysql:mysql /var/log/mysql
```
- 重启MySQL服务
```bash
service mysqld restart
```
- 建立同步账号
```bash

GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO repl@'%' IDENTIFIED BY 'xinyue';
```
- 查看状态

```bash
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      154 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

```
#### 3.2 从库配置
- /etc/my.cnf
```bash
#设定server id
server_id=102
#开启二进制日志
log_bin=/var/log/mysql/mysql-bin.log
#从库使用日志
relay_log=/var/log/mysql/mysql-relay-bin.log
#配置从库只读
read_only=1
```
- 创建二进制log目录
```bash
mkdir /var/log/mysql
chown mysql:mysql /var/log/mysql
```
- 重启MySQL服务
```bash
service mysqld restart
```
- 连接主库
```bash
mysql> CHANGE MASTER TO MASTER_HOST='192.168.12.105', MASTER_USER='repl', MASTER_PASSWORD='xinyue', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=0;
```
- 启动同步
```bash
mysql> start slave;
```
- 查看状态
```bash
mysql> show slave status \G;
#出现即为成功
Slave_IO_Running: Yes
Slave_SQL_Running: Yes

```
### 4 MySQL性能优化

- MySQL配置文件内

```bash
#最大客户端连接数
>mysql set GLOBAL max_connections = 1000;
max_connections=1000
#开启慢查询日志
slow_query_log = 1
#超出次设定值的SQL即被记录到慢查询日志
long_query_time = 6
slow_query_log_file = /var/log/docker_log/mysql/slow.log
#开启复制从库复制的慢查询的日志
log_slow_slave_statements = 1

max_connections = 1000
thread_stack = 512K

innodb_flush_log_at_trx_commit = 2
innodb_file_per_table = 1
innodb_log_file_size = 128M
innodb_log_files_in_group = 2 

innodb_log_buffer_size = 2048K
innodb_buffer_pool_size = 208M      

key_buffer_size = 208M              
read_buffer_size = 512K
read_rnd_buffer_size = 512K

sort_buffer_size = 512K
join_buffer_size = 512K
tmp_table_size = 512K


long_query_time = 5
log_output = TABLE
slow_query_log = ON

max_allowed_packet = 160M

```

## 数据导入

```bash
drop database lecai_ga1;
mysql> create database lecai_ga2;      # 创建数据库
mysql> use lecai_ga2;                  # 使用已创建的数据库 
mysql> set names utf8;           # 设置编码
mysql> source lecai_ga2.sql  # 导入备份数据库

mysql -uroot -p lecai_ga2 < lecai_ga2.sql
```

## 数据导出

```bash
#所有数据库
mysqldump -uroot –all-databases
#特定数据库
mysqldump -uroot -h 10.0.0.5 -p2o#n1f831@#SD123abc --databases lecai_ga1  > lecai_ga2.sql

mysqldump -uroot -p123456 -ntd -R --databases kuaiyun  privatecloud test yjk_client yjk_db yjk_instance > pcp.sql

#导入
CREATE DATABASE `kuaiyun` DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
#先创建数据库 kuaiyun  privatecloud test yjk_client yjk_db yjk_instance
#字符集utf8 -- UTF-8 Unicode   排序规则 utf8_general_ci
mysql -h 192.168.66.108 -uroot -pmysql1234 < pcp.sql
```

### 5.7默认时间

数据库迁移旧版本数据库时间默认值是全是0的话，在msyql5.7中应该在sql模式中取消非0限制

```bash
#修改my.cnf文件，在[mysqld]中添加
sql-mode=STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
```

查看sql-mode SELECT @@sql_mode;

## 数据迁移

```bash
#-d为阻止数据导入，仅仅导表结构
mysqldump -u user -ppass -d olddb | mysql -u user -ppass -D newdb
```
