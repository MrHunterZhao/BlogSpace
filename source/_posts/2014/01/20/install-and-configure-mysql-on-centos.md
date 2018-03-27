---
title: CentOS通过yum安装配置MySQL
date: 2014-01-20 12:06:34
categories: MySQL
tags:
    - MySQL
    - CentOS
---

# 安装MySQL
```bash
[root@HunterWorkStation local]# yum -y install mysql mysql-server
```


# 启动MySQL服务
```bash
[root@HunterWorkStation local]# service mysqld start

Initializing MySQL database:  Installing MySQL system tables...    OK
Filling help tables...   OK

To start mysqld at boot time you have to copy
support-files/mysql.server to the right place for your system

PLEASE REMEMBER TO SET A PASSWORD FOR THE MySQL root USER !
To do so, start the server, then issue the following commands:

/usr/bin/mysqladmin -u root password 'new-password'
/usr/bin/mysqladmin -u root -h localhost.localdomain password 'new-password'

Alternatively you can run:
/usr/bin/mysql_secure_installation

which will also give you the option of removing the test
databases and anonymous user created by default.  This is
strongly recommended for production servers.

See the manual for more instructions.
You can start the MySQL daemon with:

cd /usr ; /usr/bin/mysqld_safe &
You can test the MySQL daemon with mysql-test-run.pl
cd /usr/mysql-test ; perl mysql-test-run.pl

Please report any problems with the /usr/bin/mysqlbug script!


Starting mysqld:                                           [  OK  ]
```
<!-- more -->
# 设置root用户密码
```bash
[root@HunterWorkStation local]# mysqladmin -u root password 'YOUR PASSWORD HERE'
```

# 登陆数据库并授权
此时用户是无法远程登陆该数据库的，接下来为用户授权
```bash
[root@HunterWorkStation local]# mysql -u root -p

Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 4
Server version: 5.1.71 Source distribution

Copyright (c) 2000, 2013, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> grant all on *.* to root@'%' identified by '123456';  # 为所有主机授权使用root用户操作所有库和表，密码123456
Query OK, 0 rows affected (0.00 sec)
```


# 设置MySQL服务开机自启动
```bash
[root@HunterWorkStation local]# chkconfig mysqld on
```

设置后查看是否设置成功
```bash
[root@HunterWorkStation local]# chkconfig --list | grep mysqld

mysqld    0:off	 1:off  2:on  3:on  4:on  5:on  6:off   # 2-5为on则表示自启动
```

# 设置服务端编码
```bash
[root@HunterWorkStation local]# vim /etc/my.cnf

[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
user=mysql
default-character-set=utf8
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
```

# 重启MySQL服务
```bash
[root@HunterWorkStation local]# service mysqld restart
Stopping mysqld:                                           [  OK  ]
Starting mysqld:                                           [  OK  ]
```


至此，安装配置完毕，接来下可以用客户端连接操作MySQL。