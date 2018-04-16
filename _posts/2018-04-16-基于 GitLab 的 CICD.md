---
layout:     post
title:      "基于 GitLab 的 CICD"
date:       2018-04-16
author:     "pggsnap"
tags:
    - GitLab
    - CI
    - Docker
---

# 目标
程序改动 push 到 GitLab 时，自动编译并部署。

# 环境
|IP|角色|
| --  | -- |
|192.168.99.2|本机|
|192.168.99.16|GitLab & GitLab Runner|
|192.168.99.15|Docker 私有仓库|
|192.168.99.17|应用服务器|

## 安装 GitLab，Docker
192.168.99.16 安装 GitLab，Docker，略。

## 安装 GitLab Runner：
192.168.99.16 安装 GitLab Runner:

- 安装。`docker run -d --name gitlab-runner --restart always -v /var/run/docker.sock:/var/run/docker.sock -v /etc/gitlab-runner/config.toml:/etc/gitlab-runner/config.toml gitlab/gitlab-runner:alpine-v10.6.0`
- 注册项目到 GitLab Runner。`docker exec -it gitlab-runner gitlab-ci-multi-runner register` 根据提示填写项目的 url，token 等信息。
- 根据实际情况，修改 /etc/gitlab-runner/config.toml 的内容。比如我的修改完之后如下：
    ```
    [root@gitlab root]# cat /etc/gitlab-runner/config.toml
    concurrent = 1
    check_interval = 0

    [[runners]]
      name = "gitlab-cicd-demo"
      url = "http://192.168.99.16/"
      token = "74948a1ba7013c546d6aa14e03211a"
      executor = "docker"
      [runners.docker]
        tls_verify = false
        image = "centos"
        privileged = false
        disable_cache = false
        volumes = ["/cache","/var/run/docker.sock:/var/run/docker.sock","/root/.m2:/root/.m2","/data/root:/root"]
        pull_policy = "if-not-present"
        shm_size = 0
      [runners.cache]
    ```
    volumes 新增了 .m2 目录的映射关系，java 程序可以从宿主机下载依赖；新增了 /data/root 的映射关系，用于保存打包的 jar 文件。
- 之后需要重启容器。`docker restart gitlab-runner`

# GitLab CI 配置

## Dockerfile
运行一个 java 程序，前提是将程序打包成 jar 包并拷贝到 Dockerfile 文件相同目录。
```
FROM java:8-jdk-alpine
MAINTAINER pggsnap

COPY gitlab-cicd-demo.jar /gitlab-cicd-demo.jar
CMD ["/bin/sh", "-c", "java -XX:+PrintFlagsFinal -XX:+PrintGCDetails $JAVA_OPTIONS -jar /gitlab-cicd-demo.jar"]
```

## .gitlab-ci.yml
完成程序的自动打包，并生成镜像。
```
before_script:  # debug 时可以开启，方便调试
  - pwd
  - env

stages:
  - package
  - build

pre-package:
  image: maven:3-jdk-8
  stage: package
  script:
    - "mvn clean package"
    - "cp target/*.jar /root/gitlab-cicd-demo.jar"
  only:
    - master

pre-build:
  image: docker:18.04
  stage: build
  script:
    - "cp /root/gitlab-cicd-demo.jar ./"
    - "docker build -t gitlab-cicd-demo:0.0.1 ."
  only:
    - master
```
流程如下：
- 通过基于 maven:3-jdk-8 镜像生成的容器执行 mvn 命令，生成 jar 包后拷贝到容器 /root/gitlab-cicd-demo.jar，由于 /etc/gitlab-runner/config.toml 文件中设置了文件映射，因此容器中的 jar 包会拷贝到宿主机的 /data/root 目录中。
- 基于 docker:18.04 镜像生成的容器会拷贝来自宿主机 /data/root 目录中的 jar 包到容器的 /root 目录，之后拷贝该 jar 包到 Dockerfile 文件所在的目录后，通过 `docker build` 命令构建镜像。

# 测试

- 本机创建 gitlab-cicd-demo 工程，随意更改代码后 push 到 GitLab。
- 登陆到 GitLab 主页查看 pipelines 是否成功。
- 成功之后就可以在宿主机（192.168.99.16）上查看镜像是否成功构建。
    ```
    [root@gitlab root]# docker image ls
    REPOSITORY                    TAG                 IMAGE ID            CREATED             SIZE
    gitlab-cicd-demo              0.0.1               132067862ffa        5 hours ago         164MB
    ```
- 根据该镜像运行容器。`docker run -d --name gitlab-cicd-demo-0.0.1 -p 12345:12345 -e JAVA_OPTIONS='-Xmx512m' gitlab-cicd-demo:0.0.1`
- 测试接口。
    ```
    [root@gitlab root]# curl localhost:12345
    hello-world
    ```
