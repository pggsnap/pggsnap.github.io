---
layout:     post
title:      "JProfiler 监控远程服务器上的 Java 应用"
date:       2018-07-19
author:     "pggsnap"
tags:
    - JProfiler
---

本地 Windows 系统安装 JProfiler 9.2，需要监控远端主机 10.201.0.28 上的 Java 程序。

# 远程主机下载 rpm 包进行安装
```
[root@gateway opt]# rpm -ivh jprofiler_linux_9_2_1.rpm
```

# 远程主机选择需要监控的 java 程序
```
[root@gateway bin]# pwd
/opt/jprofiler9/bin
[root@gateway bin]# jpenable
Select a JVM:
glsc-job-executor-sample-springbo...spring.profiles.active=test [2643] [1]
ms-gateway-0.0.1-SNAPSHOT-2.0.jar --spring.profiles.active=test [4711] [2]
ms-gateway-0.0.1-SNAPSHOT.jar --s...e=test -Xms2048m -Xmx2048m [16791] [3]
org.elasticsearch.bootstrap.Elasti.../elasticsearch.pid --quiet [1144] [4]
3
Please select the profiling mode:
GUI mode (attach with JProfiler GUI) [1, Enter]
Offline mode (use config file to set profiling settings) [2]
1
Please enter a profiling port
[44968]

You can now use the JProfiler GUI to connect on port 44968
```

# 连接远程主机

1. 本机安装 JProfiler。

2. 新建到远端应用的连接。

![](/blog_img/jprofiler-use-1.jpg)

然后根据提示一路 next 下去：
![](/blog_img/jprofiler-use-2.jpg)

![](/blog_img/jprofiler-use-3.jpg)

![](/blog_img/jprofiler-use-4.jpg)

![](/blog_img/jprofiler-use-5.jpg)

![](/blog_img/jprofiler-use-6.jpg)

![](/blog_img/jprofiler-use-7.jpg)

![](/blog_img/jprofiler-use-8.jpg)

![](/blog_img/jprofiler-use-9.jpg)
