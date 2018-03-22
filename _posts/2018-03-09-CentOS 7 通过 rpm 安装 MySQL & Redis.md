---
layout:     post
title:      "CentOS 7 通过 rpm 安装 MySQL & Redis"
date:       2018-03-09
author:     "pggsnap"
tags:
    - CentOS
    - MySQL
    - Redis
---

# 安装 MySQL

1. 卸载 mariadb。

```
[root@localhost opt]# rpm -qa | grep mariadb
mariadb-libs-5.5.52-1.el7.x86_64
[root@localhost opt]# rpm -e --nodeps mariadb-libs-5.5.52-1.el7.x86_64
[root@localhost opt]#
```

2. 下载 MySQL 并保存在 /opt/mysql 目录下。
> [官网](https://cdn.mysql.com//Downloads/MySQL-5.7/mysql-5.7.21-1.el7.x86_64.rpm-bundle.tar)

3. 解压安装 MySQL。

```
[root@localhost opt]# cd mysql/
[root@localhost mysql]# tar -xvf mysql-5.7.21-1.el7.x86_64.rpm-bundle.tar
[root@localhost mysql]# ll
total 1160048
-rw-r--r-- 1 root root  593940480 Mar  9 09:45 mysql-5.7.21-1.el7.x86_64.rpm-bundle.tar
-rw-r--r-- 1 7155 31415  25107316 Dec 28 20:53 mysql-community-client-5.7.21-1.el7.x86_64.rpm
-rw-r--r-- 1 7155 31415    278844 Dec 28 20:53 mysql-community-common-5.7.21-1.el7.x86_64.rpm
-rw-r--r-- 1 7155 31415   3779988 Dec 28 20:53 mysql-community-devel-5.7.21-1.el7.x86_64.rpm
-rw-r--r-- 1 7155 31415  46256768 Dec 28 20:53 mysql-community-embedded-5.7.21-1.el7.x86_64.rpm
-rw-r--r-- 1 7155 31415  24078148 Dec 28 20:53 mysql-community-embedded-compat-5.7.21-1.el7.x86_64.rpm
-rw-r--r-- 1 7155 31415 128571868 Dec 28 20:53 mysql-community-embedded-devel-5.7.21-1.el7.x86_64.rpm
-rw-r--r-- 1 7155 31415   2238596 Dec 28 20:53 mysql-community-libs-5.7.21-1.el7.x86_64.rpm
-rw-r--r-- 1 7155 31415   2115904 Dec 28 20:54 mysql-community-libs-compat-5.7.21-1.el7.x86_64.rpm
-rw-r--r-- 1 7155 31415  55662616 Dec 28 20:54 mysql-community-minimal-debuginfo-5.7.21-1.el7.x86_64.rpm
-rw-r--r-- 1 7155 31415 171890056 Dec 28 20:54 mysql-community-server-5.7.21-1.el7.x86_64.rpm
-rw-r--r-- 1 7155 31415  15289580 Dec 28 20:54 mysql-community-server-minimal-5.7.21-1.el7.x86_64.rpm
-rw-r--r-- 1 7155 31415 118654584 Dec 28 20:54 mysql-community-test-5.7.21-1.el7.x86_64.rpm
[root@localhost mysql]#
[root@localhost mysql]# rpm -ivh mysql-community-common-5.7.21-1.el7.x86_64.rpm
warning: mysql-community-common-5.7.21-1.el7.x86_64.rpm: Header V3 DSA/SHA1 Signature, key ID 5072e1f5: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:mysql-community-common-5.7.21-1.e################################# [100%]
[root@localhost mysql]#
[root@localhost mysql]# rpm -ivh mysql-community-libs-5.7.21-1.el7.x86_64.rpm
warning: mysql-community-libs-5.7.21-1.el7.x86_64.rpm: Header V3 DSA/SHA1 Signature, key ID 5072e1f5: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:mysql-community-libs-5.7.21-1.el7################################# [100%]
[root@localhost mysql]#
[root@localhost mysql]# rpm -ivh mysql-community-client-5.7.21-1.el7.x86_64.rpm
warning: mysql-community-client-5.7.21-1.el7.x86_64.rpm: Header V3 DSA/SHA1 Signature, key ID 5072e1f5: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:mysql-community-client-5.7.21-1.e################################# [100%]
[root@localhost mysql]#
[root@localhost mysql]# rpm -ivh mysql-community-server-5.7.21-1.el7.x86_64.rpm
warning: mysql-community-server-5.7.21-1.el7.x86_64.rpm: Header V3 DSA/SHA1 Signature, key ID 5072e1f5: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:mysql-community-server-5.7.21-1.e################################# [100%]
```

4. 配置 MySQL。

```
[root@localhost mysql]# mysqld --initialize --user=mysql
[root@localhost mysql]#
[root@localhost mysql]# cat /var/log/mysqld.log | grep password
2018-03-09T02:08:52.404878Z 1 [Note] A temporary password is generated for root@localhost: kehA5GYrR>dB
[root@localhost mysql]#
[root@localhost mysql]# systemctl start mysqld
[root@localhost mysql]# systemctl status mysqld
● mysqld.service - MySQL Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2018-03-09 10:09:58 CST; 4s ago
     Docs: man:mysqld(8)
           http://dev.mysql.com/doc/refman/en/using-systemd.html
  Process: 15991 ExecStart=/usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid $MYSQLD_OPTS (code=exited, status=0/SUCCESS)
  Process: 15969 ExecStartPre=/usr/bin/mysqld_pre_systemd (code=exited, status=0/SUCCESS)
 Main PID: 15994 (mysqld)
   CGroup: /system.slice/mysqld.service
           └─15994 /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid

Mar 09 10:09:46 localhost.localdomain systemd[1]: Starting MySQL Server...
Mar 09 10:09:58 localhost.localdomain systemd[1]: Started MySQL Server.
[root@localhost mysql]#
[root@localhost mysql]# mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.7.21

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> set password = password('Uu12%#hyt');
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> exit
Bye
```

5. 设置 MySQL 开机自启动。

```
[root@localhost mysql]# systemctl enable mysqld
```

# 安装 Redis

1. 安装依赖 GCC, GCC-C++，具体文件见下图。

```
[root@localhost redis]# cd redis-lib/
[root@localhost redis-lib]# ll
total 47640
-rw-r--r-- 1 root root    67624 Mar  9 11:14 autogen-libopts-5.18-5.el7.x86_64.rpm
-rw-r--r-- 1 root root  6230356 Mar  9 11:14 cpp-4.8.5-16.el7.x86_64.rpm
-rw-r--r-- 1 root root 16960476 Mar  9 11:14 gcc-4.8.5-16.el7.x86_64.rpm
-rw-r--r-- 1 root root  7519364 Mar  9 11:14 gcc-c++-4.8.5-16.el7.x86_64.rpm
-rw-r--r-- 1 root root  1111876 Mar  9 11:14 glibc-devel-2.17-196.el7.x86_64.rpm
-rw-r--r-- 1 root root   691620 Mar  9 11:14 glibc-headers-2.17-196.el7.x86_64.rpm
-rw-r--r-- 1 root root  6247648 Mar  9 11:14 kernel-headers-3.10.0-693.el7.x86_64.rpm
-rw-r--r-- 1 root root    38232 Mar  9 11:14 keyutils-libs-devel-1.5.8-3.el7.x86_64.rpm
-rw-r--r-- 1 root root   272876 Mar  9 11:14 krb5-devel-1.15.1-8.el7.x86_64.rpm
-rw-r--r-- 1 root root    31400 Mar  9 11:14 libcom_err-devel-1.42.9-10.el7.x86_64.rpm
-rw-r--r-- 1 root root   100504 Mar  9 11:14 libgcc-4.8.5-16.el7.x86_64.rpm
-rw-r--r-- 1 root root   157600 Mar  9 11:14 libgomp-4.8.5-16.el7.x86_64.rpm
-rw-r--r-- 1 root root    51732 Mar  9 11:14 libmpc-1.0.1-3.el7.x86_64.rpm
-rw-r--r-- 1 root root   190704 Mar  9 11:14 libselinux-devel-2.5-11.el7.x86_64.rpm
-rw-r--r-- 1 root root    75980 Mar  9 11:14 libsepol-devel-2.5-6.el7.x86_64.rpm
-rw-r--r-- 1 root root   308312 Mar  9 11:14 libstdc++-4.8.5-16.el7.x86_64.rpm
-rw-r--r-- 1 root root  1576488 Mar  9 11:14 libstdc++-devel-4.8.5-16.el7.x86_64.rpm
-rw-r--r-- 1 root root    11776 Mar  9 11:14 libverto-devel-0.2.5-4.el7.x86_64.rpm
-rw-r--r-- 1 root root   208316 Mar  9 11:14 mpfr-3.1.1-4.el7.x86_64.rpm
-rw-r--r-- 1 root root   560368 Mar  9 11:14 ntp-4.2.6p5-25.el7.centos.2.x86_64.rpm
-rw-r--r-- 1 root root   812020 Mar  9 11:14 openssl098e-0.9.8e-29.el7.centos.3.x86_64.rpm
-rw-r--r-- 1 root root   503460 Mar  9 11:14 openssl-1.0.2k-8.el7.x86_64.rpm
-rw-r--r-- 1 root root  1577900 Mar  9 11:14 openssl-devel-1.0.2k-8.el7.x86_64.rpm
-rw-r--r-- 1 root root  1248348 Mar  9 11:14 openssl-libs-1.0.2k-8.el7.x86_64.rpm
-rw-r--r-- 1 root root    54928 Mar  9 11:14 pkgconfig-0.27.1-4.el7.x86_64.rpm
-rw-r--r-- 1 root root  1980564 Mar  9 11:14 tcl-8.5.13-8.el7.x86_64.rpm
-rw-r--r-- 1 root root    91872 Mar  9 11:14 zlib-1.2.7-17.el7.x86_64.rpm
-rw-r--r-- 1 root root    51044 Mar  9 11:14 zlib-devel-1.2.7-17.el7.x86_64.rpm
[root@localhost redis-lib]#
[root@localhost redis-lib]# rpm -Uvh *.rpm --nodeps --force
[root@localhost redis-lib]# gcc -v
[root@localhost redis-lib]# g++ -v
```

2. 下载 Redis 并保存到 /opt/redis 目录下。
> [官网](http://download.redis.io/releases/redis-4.0.8.tar.gz)

3. 解压安装 Redis。

```
[root@localhost redis]# tar xzf redis-4.0.8.tar.gz
[root@localhost redis]# cd redis-4.0.8
[root@localhost redis-4.0.8]# make MALLOC=libc
[root@localhost redis-4.0.8]# make install
```

4. 配置 Redis。

```
[root@localhost redis-4.0.8]# mkdir -p /etc/redis && cp redis.conf /etc/redis
[root@localhost redis-4.0.8]# vim /etc/redis/redis.conf
daemonize no 修改为 daemonize yes
[root@localhost redis-4.0.8]# /usr/local/bin/redis-server /etc/redis/redis.conf # 启动 redis
[root@localhost redis-4.0.8]# redis-cli
```