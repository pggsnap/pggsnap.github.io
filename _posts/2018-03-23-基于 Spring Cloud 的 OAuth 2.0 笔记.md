---
layout:     post
title:      "基于 Spring Cloud 的 OAuth 2.0 笔记"
date:       2018-03-23
author:     "pggsnap"
tags:
    - Spring Cloud
    - OAuth
    - Spring Security
---

# 需求描述

实现一个 OAuth 2.0 认证的微服务 ms-auth，其作用是对于外部访问（通过 zuul 网关过来的请求），需要先认证授权，通过后网关才能转发请求；对于内部微服务之间的访问，则无需认证。

# OAuth 2.0

关于 OAuth 2.0 的理解以及一些概念，可以参考:  

> [理解OAuth 2.0](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)  
> [从零开始的Spring Security Oauth2](http://blog.didispace.com/spring-security-oauth2-xjf-1/)  

本文采用的是密码模式。

### 认证服务器

```
@Configuration
@EnableAuthorizationServer
public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {

    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private RedisConnectionFactory connectionFactory;

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        // 配置两个客户端, 一个用于 password 认证, 一个用于 client 认证
        clients.inMemory()
                .withClient("server")
                .secret("server")
                .authorizedGrantTypes("password", "refresh_token")
                .scopes("x")    // 这边指定的 scopes 没啥作用，但是不设置的话会报错
                .and().withClient("in")
                .secret("in")
                .authorizedGrantTypes("client_credentials", "refresh_token")
                .scopes("x");
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) {
        endpoints.authenticationManager(authenticationManager)
                .tokenStore(tokenStore());  // token 存储在 redis 中
    }

    @Bean
    public TokenStore tokenStore() {
        return new RedisTokenStore(connectionFactory);
    }
}
```

### 资源服务器

```
@Configuration
@EnableResourceServer
public class ResourceServerConfig extends ResourceServerConfigurerAdapter {
    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.csrf().disable()
                .exceptionHandling()
                .authenticationEntryPoint((request, response, authException) -> response.sendError(HttpServletResponse.SC_UNAUTHORIZED))
                .and()
                .authorizeRequests()
                .anyRequest().authenticated()
                .and()
                .httpBasic();
    }
}
```

### Spring Security

```
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Bean
    public UserDetailsService userDetailsService() {
        return new MyUserDetailService();   // 自定义 UserDetailService，实现简单的用户-权限逻辑，相关数据存储在 MySQL
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService()).passwordEncoder(passwordEncoder());
    }
}
```

# 实现方案讨论

### 通过 Spring Security 的注解

常见的做法是在需要认证的接口上，设置相关注解，比如：

```
@PreAuthorize("hasAuthority('hello')")  // 访问该接口需要拥有 hello 的权限
@RequestMapping(value = "/hello", method = RequestMethod.GET)
public String hello(String name) {
    return "hello, " + name;
}
```

- 这样做的好处是从代码层面上看，逻辑比较清晰，哪些接口需要认证，需要什么样的权限，哪些接口不需要认证一目了然。  
- 缺点是:
    - 该接口要么需要认证，要么不需要认证。如果内部服务调用无需认证，而外部调用需要认证的话，无法实现。  
    - 权限关系和代码耦合。如果有些接口需要变更权限或者无需认证了，需要更改代码并重新部署服务才能生效。

### 通过 zuul 网关过滤功能

如果把请求的授权认证放在网关处理，内部微服务不配置 Spring Security 等认证的话，那么就可以实现外部请求需要认证授权，而内部调用无需认证的需求了。

- 对于需要 OAuth 2.0 认证的外部请求，其请求头中必须包含 token 信息，可以根据该信息知道用户名，进而查询所拥有的权限（通过 UserDetailService 的 loadUserByUsername 方法）。

- 管理接口和所需权限的逻辑关系。  
    创建 table: privileges，属性包括 project_name（服务名称），api（接口路径），privilege（访问所需权限）。
    获取外部请求中的 servletPath，通过查表，就可以确定访问该接口所需要的权限了。

    ```
    +----+--------------+------------------+-----------------+
    | id | project_name | api              | privilege       |
    +----+--------------+------------------+-----------------+
    |  1 | ms-server    | /ms-server/hello | ms-server:hello |
    +----+--------------+------------------+-----------------+
    ```

- 通过请求中带有的 token 以及请求路径等信息，确定访问该接口需要的权限以及用户拥有的权限进行比较，从而实现了对外部请求的认证需求。ms-auth 微服务可以实现一个接口（入参为 token 以及 api），提供给网关使用；如果认证通过，则网关转发请求；如果认证失败，则直接拒绝。  
ms-auth 统一维护 MySQL 以及 Redis，可以在服务启动时将表 privileges 的信息加载到 Redis 中去，这样可以直接在 Redis 中获取需要的所有信息。具体流程如图所示：
![](/blog_img/2018032301.jpg)

# 测试

- 创建一个微服务 ms-server 用于测试，提供了 hello 接口，该接口需要认证（在数据库中配置）。

```
@RestController
public class HelloController {
    @RequestMapping(value = "/hello", method = RequestMethod.GET)
    public String hello(String name) {
        return "hello, " + name;
    }
```

- 调用 ms-auth 认证接口，获取 token。
![](/blog_img/2018032302.jpg)

- 通过网关调用 hello 接口。
![](/blog_img/2018032303.jpg)

# 一些细节
