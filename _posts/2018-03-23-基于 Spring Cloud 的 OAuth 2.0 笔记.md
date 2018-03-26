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

- token 在 Redis 中的存储
> [spring security oauth2使用redis存储token](https://segmentfault.com/a/1190000012353924)

![](/blog_img/2018032304.jpg)
一共有 9 个 key，可以通过以下命令查看:

```
127.0.0.1:6379> keys *
 1) "uname_to_access:server:pggsnap"
 2) "access:a89a6efc-9e49-4973-9227-926e2d8b40b3"
 3) "refresh_to_access:458c8993-305c-44f4-aace-b9d233110b22"
 4) "auth_to_access:2bed885bad976e388f1dd9a3012727c4"
 5) "access_to_refresh:a89a6efc-9e49-4973-9227-926e2d8b40b3"
 6) "client_id_to_access:server"
 7) "refresh_auth:458c8993-305c-44f4-aace-b9d233110b22"
 8) "auth:a89a6efc-9e49-4973-9227-926e2d8b40b3"
 9) "refresh:458c8993-305c-44f4-aace-b9d233110b22"
 127.0.0.1:6379> type "auth_to_access:2bed885bad976e388f1dd9a3012727c4"
string
127.0.0.1:6379> get "auth_to_access:2bed885bad976e388f1dd9a3012727c4"
"\xac\xed\x00\x05sr\x00Corg.springframework.security.oauth2.common.DefaultOAuth2AccessToken\x0c\xb2\x9e6\x1b$\xfa\xce\x02\x00\x06L\x00\x15additionalInformationt\x00\x0fLjava/util/Map;L\x00\nexpirationt\x00\x10Ljava/util/Date;L\x00\x0crefreshTokent\x00?Lorg/springframework/security/oauth2/common/OAuth2RefreshToken;L\x00\x05scopet\x00\x0fLjava/util/Set;L\x00\ttokenTypet\x00\x12Ljava/lang/String;L\x00\x05valueq\x00~\x00\x05xpsr\x00\x1ejava.util.Collections$EmptyMapY6\x14\x85Z\xdc\xe7\xd0\x02\x00\x00xpsr\x00\x0ejava.util.Datehj\x81\x01KYt\x19\x03\x00\x00xpw\b\x00\x00\x01bev\xad\xc4xsr\x00Lorg.springframework.security.oauth2.common.DefaultExpiringOAuth2RefreshToken/\xdfGc\x9d\xd0\xc9\xb7\x02\x00\x01L\x00\nexpirationq\x00~\x00\x02xr\x00Dorg.springframework.security.oauth2.common.DefaultOAuth2RefreshTokens\xe1\x0e\ncT\xd4^\x02\x00\x01L\x00\x05valueq\x00~\x00\x05xpt\x00$458c8993-305c-44f4-aace-b9d233110b22sq\x00~\x00\tw\b\x00\x00\x01b\xfa\xcf\x19\xc3xsr\x00%java.util.Collections$UnmodifiableSet\x80\x1d\x92\xd1\x8f\x9b\x80U\x02\x00\x00xr\x00,java.util.Collections$UnmodifiableCollection\x19B\x00\x80\xcb^\xf7\x1e\x02\x00\x01L\x00\x01ct\x00\x16Ljava/util/Collection;xpsr\x00\x17java.util.LinkedHashSet\xd8l\xd7Z\x95\xdd*\x1e\x02\x00\x00xr\x00\x11java.util.HashSet\xbaD\x85\x95\x96\xb8\xb74\x03\x00\x00xpw\x0c\x00\x00\x00\x10?@\x00\x00\x00\x00\x00\x01t\x00\x01xxt\x00\x06bearert\x00$a89a6efc-9e49-4973-9227-926e2d8b40b3"
```


- access_token 以及 refresh_token 的刷新

access_token 过期，refresh_token 未过期，调用刷新 token 接口，可以重新获取 access_token；
如果都过期，需要通过账户密码验证重新获取 token。

    - 获取 token

    ```
    pggsnap@mbp ~$curl -X POST -u "server:server" -d "grant_type=password&username=pggsnap&password=123456" "http://localhost:8080/ms-auth/oauth/token"
    {"access_token":"8b3de5a9-8fe2-4742-9157-c9871bc23d0e","token_type":"bearer","refresh_token":"a9c02ab1-4c41-4aca-92f3-810f4686c998","expires_in":119,"scope":"x"}
    ```

    - 刷新 token

    ```
    pggsnap@mbp ~$curl -X POST -u "server:server" -d "grant_type=refresh_token&refresh_token=a9c02ab1-4c41-4aca-92f3-810f4686c998" "http://localhost:8080/ms-auth/oauth/token"
    {"access_token":"58533360-54d4-4e5c-b5f4-d8d57590114f","token_type":"bearer","refresh_token":"a9c02ab1-4c41-4aca-92f3-810f4686c998","expires_in":119,"scope":"x"}
    ```


完整代码参考： [GitHub](https://github.com/pggsnap/oauth)