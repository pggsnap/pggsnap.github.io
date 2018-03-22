---
layout:     post
title:      "卸载 OpenJDK 并安装 Oracle JDK"
date:       2018-03-08
author:     "pggsnap"
tags:
    - CentOS
    - Java
---

## 卸载OpenJDK

```
[root@localhost ~]# rpm -qa | grep jdk
java-1.8.0-openjdk-1.8.0.161-0.b14.el7_4.x86_64
java-1.8.0-openjdk-headless-1.8.0.161-0.b14.el7_4.x86_64
java-1.7.0-openjdk-1.7.0.161-2.6.12.0.el7_4.x86_64
java-1.7.0-openjdk-headless-1.7.0.161-2.6.12.0.el7_4.x86_64
copy-jdk-configs-2.2-5.el7_4.noarch
[root@localhost ~]#
[root@localhost ~]# rpm -e --nodeps java-1.8.0-openjdk-1.8.0.161-0.b14.el7_4.x86_64 java-1.8.0-openjdk-headless-1.8.0.161-0.b14.el7_4.x86_64 java-1.7.0-openjdk-1.7.0.161-2.6.12.0.el7_4.x86_64 java-1.7.0-openjdk-headless-1.7.0.161-2.6.12.0.el7_4.x86_64
[root@localhost ~]#
[root@localhost ~]# java -version
-bash: /usr/bin/java: No such file or directory
```

### 安装Oracle JDK

- [官网](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)下载并保存在/opt/目录下。

- 安装
```
[root@localhost ~]# cd /opt/
[root@localhost opt]# ll
total 170124
-rw-r--r--  1 root root 174204631 Mar  8 17:19 jdk-8u162-linux-x64.rpm
drwxr-xr-x. 2 root root         6 Sep  7 07:11 rh
[root@localhost opt]# rpm -ivh jdk-8u162-linux-x64.rpm
[root@localhost opt]# java -version
java version "1.8.0_162"
Java(TM) SE Runtime Environment (build 1.8.0_162-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.162-b12, mixed mode)
```