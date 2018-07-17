---
layout:     post
title:      "Spring Cloud 系列之 Zuul 限流"
date:       2018-07-17
author:     "pggsnap"
tags:
    - Spring Cloud
    - Zuul
    - 限流
---

# 理解 spring-cloud-zuul-ratelimit

```xml
<dependency>
    <groupId>com.marcosbarbero.cloud</groupId>
    <artifactId>spring-cloud-zuul-ratelimit</artifactId>
    <version>2.0.2.RELEASE</version>
</dependency>
```

## 原理

基于 zuul 网关的过滤功能，新增 RateLimitPreFilter（order：-1） 以及 RateLimitPostFilter（order：990） 过滤器。

在内存或者缓存或者数据库中维护一个 Map，根据请求以及限流粒度生成 key，接收到新的请求时，value 值加 1。和限流策略中的 limit 或者 quota 对比，如果超出则报错。

## 限流粒度
```java
public static enum Type {
    ORIGIN,
    USER,
    URL;

    private Type() {
    }
}
```

- ORIGIN：基于 ip 地址；

- URL： 基于请求的 url；

- USER：基于用户限流，如果项目整合了 Spring Security 安全框架，能够根据请求头中的 Authorization 获取真实的调用方（SecurityContextHolderAwareRequestWrapper.getRemoteUser），如果没有获取到，设置 user 为 anonymous。

默认支持以上 3 种粒度，可以组合使用，根据限流粒度生成 key 的关键代码如下：
```java
public class DefaultRateLimitKeyGenerator implements RateLimitKeyGenerator {

    public String key(final HttpServletRequest request, final Route route, final Policy policy) {
        StringJoiner joiner = new StringJoiner(":");
        joiner.add(this.properties.getKeyPrefix());     // 默认为 ${spring.application.name}，比如 gateway
        if (route != null) {
            joiner.add(route.getId());  // 增加路由信息，key 变更为：gateway:api-dw
        }

        policy.getType().forEach((matchType) -> {
            if (route != null && Type.URL.equals(matchType.getType())) {
                joiner.add(route.getPath());    // 如果开启了 url 粒度，那么增加 url 信息，key 格式是这样: gateway:api-dw:/dw-service/test
                this.addMatcher(joiner, matchType);
            }

            if (Type.ORIGIN.equals(matchType.getType())) {
                joiner.add(this.rateLimitUtils.getRemoteAddress(request));  // 如果开启了 origin 粒度，增加 ip 信息，key 格式：gateway:api-dw:/dw-service/test:10.10.10.10:
                this.addMatcher(joiner, matchType);
            }

            if (Type.USER.equals(matchType.getType())) {
                joiner.add(this.rateLimitUtils.getUser(request));   // 如果开启了 user 粒度，增加 user 信息，key 格式：gateway:api-dw:/dw-service/test:10.10.10.10:anonymous
                this.addMatcher(joiner, matchType);
            }

        });
        return joiner.toString();
    }
}
```

如果不能满足需求，可以自定义 RateLimitKeyGenerator 实现。

## 存储方式

保存在一个时间窗口内针对（url，user，ip 以及自定义粒度）的调用次数，主要为内存、缓存以及数据库等，具体支持存储方式如下：
```java
public static enum Repository {
    REDIS,
    CONSUL,
    JPA,
    BUCKET4J_JCACHE,
    BUCKET4J_HAZELCAST,
    BUCKET4J_IGNITE,
    BUCKET4J_INFINISPAN,
    IN_MEMORY;

    private Repository() {
    }
}
```

## 限流策略
```yaml
default-policy:
  limit: 20             # 单位时间内允许访问的次数，20 次
  quota: 20             # 单位时间内允许访问的总时间（统计每次请求的时间综合）, 20s
  refresh-interval: 60  # 单位时间设置，60s
  type:                 # 限流粒度：url + user 组合形式
    - url
    - user
```

以上配置意思是：在一个时间窗口 60s 内，最多允许 20 次访问，或者总请求时间小于 20s。相关代码参考：
```java
public Object run() {
    RequestContext ctx = RequestContext.getCurrentContext();
    HttpServletResponse response = ctx.getResponse();
    HttpServletRequest request = ctx.getRequest();
    Route route = this.route(request);
    this.policy(route, request).forEach((policy) -> {
        String key = this.rateLimitKeyGenerator.key(request, route, policy);
        Rate rate = this.rateLimiter.consume(policy, key, (Long)null);
        String httpHeaderKey = key.replaceAll("[^A-Za-z0-9-.]", "_").replaceAll("__", "_");
        Long limit = policy.getLimit();
        Long remaining = rate.getRemaining();
        if (limit != null) {
            response.setHeader("X-RateLimit-Limit-" + httpHeaderKey, String.valueOf(limit));
            response.setHeader("X-RateLimit-Remaining-" + httpHeaderKey, String.valueOf(Math.max(remaining, 0L)));
        }

        Long quota = policy.getQuota();
        Long remainingQuota = rate.getRemainingQuota();
        if (quota != null) {
            request.setAttribute("rateLimitRequestStartTime", System.currentTimeMillis());
            response.setHeader("X-RateLimit-Quota-" + httpHeaderKey, String.valueOf(quota));
            response.setHeader("X-RateLimit-Remaining-Quota-" + httpHeaderKey, String.valueOf(TimeUnit.MILLISECONDS.toSeconds(Math.max(remainingQuota, 0L))));
        }

        response.setHeader("X-RateLimit-Reset-" + httpHeaderKey, String.valueOf(rate.getReset()));
        if (limit != null && remaining < 0L || quota != null && remainingQuota < 0L) {  // limit 和 quota 任意一个不满足就返回报错
            ctx.setResponseStatusCode(HttpStatus.TOO_MANY_REQUESTS.value());
            ctx.put("rateLimitExceeded", "true");
            ctx.setSendZuulResponse(false);
            throw new RateLimitExceededException();
        }
    });
    return null;
}
```

## 配置文件
```yaml
zuul:
  semaphore:
    max-semaphores: 200
  routes:
    api-dw:
      path: /dw-service/**
      serviceId: dw-service
      stripPrefix: false
    api-hr:
      path: /hr-service/**
      serviceId: hr-service
      stripPrefix: false
  ignored-services: '*'

  ratelimit:
    enabled: true
    repository: redis
    behind-proxy: true
    policy-list:
      api-dw:   # dw-service 微服务的限流策略，优先级高于默认策略，注意需要和 zuul.routes 下的 route-id 一致
        - limit: 3
          refresh-interval: 1
          type:
            - url
    default-policy: # 默认的限流策略
      limit: 1
      quota: 10
      refresh-interval: 1
      type:
        - url
        - user
```

限流策略的确定：根据路由寻找限流策略，如果没找到则使用默认策略。
```java
public abstract class AbstractRateLimitFilter extends ZuulFilter {

    protected List<Policy> policy(Route route, HttpServletRequest request) {
        // 根据路由查找 route-id，如果有 采用该路由的限流策略，如果没有 采用默认策略
        String routeId = (String)Optional.ofNullable(route).map(Route::getId).orElse((Object)null);
        return (List)this.properties.getPolicies(routeId).stream().filter((policy) -> {
            return this.applyPolicy(request, route, policy);
        }).collect(Collectors.toList());
    }
}

public class RateLimitProperties implements Validator {
    public List<RateLimitProperties.Policy> getPolicies(String key) {
        return StringUtils.isEmpty(key) ? this.defaultPolicyList : (List)this.policyList.getOrDefault(key, this.defaultPolicyList);
    }
}
```
