---
layout:     post
title:      "Spring Cloud 系列之 Zuul 分析: 路由"
date:       2018-05-03
author:     "pggsnap"
tags:
    - Spring Cloud
    - Zuul
---

以下分析基于 Spring Cloud 版本为 Edgware.SR3。

# 一、反向代理
```
zuul:
  ignoredPatterns: /**/admin/**
  routes:
    auth-service:
      path: /uaa/**
      serviceId: auth-service
      stripPrefix: false
      sensitiveHeaders:
```

`path: /uaa/**, serviceId: auth-service`：将 /uaa/** 路径的 url 映射到 auth-service 服务；

`stripPrefix: false`：zuul 默认会剥离前缀，设置为 false 则不剥离；

`sensitiveHeaders:`：可以设置哪些敏感的 header 属性不在服务之间传递，默认为 "Cookie", "Set-Cookie", "Authorization"。

`ignoredPatterns: /**/admin/**`：路径中包括 /admin 的 url 不映射。

在 Spring Boot 中使用 @EnableZuulProxy 注解，会默认开启 endpoint 路由接口：/routes。设置 `management.security.enabled=false`，可以通过 `curl -X GET http://localhost:8080/routes` 获取路由信息，路径后增加 `?format=details`可以获取路由的详细信息；也可以通过 `curl -X POST http://localhost:8080/routes` 强制立即刷新。

# 二、Zuul Http Client
Zuul 默认使用的 Http 客户端为 Apache Http Client。如果需要改成 OkHttpClient，设置 `ribbon.okhttp.enabled=true`。
