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

```shell
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
```

默认的序列化方式为 JdkSerializationStrategy，通过反序列化查看下这些 key 分别对应哪些信息：

```shell
client_id_to_access:server -> [
    {
        "access_token": "b6fdbf99-4259-43bb-bbe7-f3c2a0168a87",
        "token_type": "bearer",
        "refresh_token": "eda15dc0-5b4a-4bc3-92c8-ffeeb63f7ded",
        "expires_in": 31356,
        "scope": "xx"
    }
]

auth:b6fdbf99-4259-43bb-bbe7-f3c2a0168a87 -> {
    "authorities": [
        {
            "authority": "dw-service:jres-test"
        },
        {
            "authority": "ms-test2:test"
        }
    ],
    "details": null,
    "authenticated": true,
    "userAuthentication": {
        "authorities": [
            {
                "authority": "dw-service:jres-test"
            },
            {
                "authority": "ms-test2:test"
            }
        ],
        "details": {
            "grant_type": "password",
            "username": "test2.0"
        },
        "authenticated": true,
        "principal": {
            "password": null,
            "username": "test2.0",
            "authorities": [
                {
                    "authority": "dw-service:jres-test"
                },
                {
                    "authority": "ms-test2:test"
                }
            ],
            "accountNonExpired": true,
            "accountNonLocked": true,
            "credentialsNonExpired": true,
            "enabled": true
        },
        "credentials": null,
        "name": "test2.0"
    },
    "principal": {
        "password": null,
        "username": "test2.0",
        "authorities": [
            {
                "authority": "dw-service:jres-test"
            },
            {
                "authority": "ms-test2:test"
            }
        ],
        "accountNonExpired": true,
        "accountNonLocked": true,
        "credentialsNonExpired": true,
        "enabled": true
    },
    "credentials": "",
    "clientOnly": false,
    "oauth2Request": {
        "clientId": "server",
        "scope": [
            "xx"
        ],
        "requestParameters": {
            "grant_type": "password",
            "username": "test2.0"
        },
        "resourceIds": [],
        "authorities": [],
        "approved": true,
        "refresh": false,
        "redirectUri": null,
        "responseTypes": [],
        "extensions": {},
        "refreshTokenRequest": null,
        "grantType": "password"
    },
    "name": "test2.0"
}

auth_to_access:897193de5c8f9f2c0ed5b3b60e7eb9e7 -> {
    "access_token": "b6fdbf99-4259-43bb-bbe7-f3c2a0168a87",
    "token_type": "bearer",
    "refresh_token": "eda15dc0-5b4a-4bc3-92c8-ffeeb63f7ded",
    "expires_in": 31318,
    "scope": "xx"
}

uname_to_access:server:test2.0 -> [
    {
        "access_token": "b6fdbf99-4259-43bb-bbe7-f3c2a0168a87",
        "token_type": "bearer",
        "refresh_token": "eda15dc0-5b4a-4bc3-92c8-ffeeb63f7ded",
        "expires_in": 31292,
        "scope": "xx"
    }
]

refresh:eda15dc0-5b4a-4bc3-92c8-ffeeb63f7ded -> {
    "value": "eda15dc0-5b4a-4bc3-92c8-ffeeb63f7ded",
    "expiration": "2018-10-12T01:59:10.967+0000"
}

refresh_auth:eda15dc0-5b4a-4bc3-92c8-ffeeb63f7ded -> {
    "authorities": [
        {
            "authority": "dw-service:jres-test"
        },
        {
            "authority": "ms-test2:test"
        }
    ],
    "details": null,
    "authenticated": true,
    "userAuthentication": {
        "authorities": [
            {
                "authority": "dw-service:jres-test"
            },
            {
                "authority": "ms-test2:test"
            }
        ],
        "details": {
            "grant_type": "password",
            "username": "test2.0"
        },
        "authenticated": true,
        "principal": {
            "password": null,
            "username": "test2.0",
            "authorities": [
                {
                    "authority": "dw-service:jres-test"
                },
                {
                    "authority": "ms-test2:test"
                }
            ],
            "accountNonExpired": true,
            "accountNonLocked": true,
            "credentialsNonExpired": true,
            "enabled": true
        },
        "credentials": null,
        "name": "test2.0"
    },
    "principal": {
        "password": null,
        "username": "test2.0",
        "authorities": [
            {
                "authority": "dw-service:jres-test"
            },
            {
                "authority": "ms-test2:test"
            }
        ],
        "accountNonExpired": true,
        "accountNonLocked": true,
        "credentialsNonExpired": true,
        "enabled": true
    },
    "credentials": "",
    "clientOnly": false,
    "oauth2Request": {
        "clientId": "server",
        "scope": [
            "xx"
        ],
        "requestParameters": {
            "grant_type": "password",
            "username": "test2.0"
        },
        "resourceIds": [],
        "authorities": [],
        "approved": true,
        "refresh": false,
        "redirectUri": null,
        "responseTypes": [],
        "extensions": {},
        "refreshTokenRequest": null,
        "grantType": "password"
    },
    "name": "test2.0"
}

refresh_to_access:eda15dc0-5b4a-4bc3-92c8-ffeeb63f7ded ->
	b6fdbf99-4259-43bb-bbe7-f3c2a0168a87

access:b6fdbf99-4259-43bb-bbe7-f3c2a0168a87 -> {
    "access_token": "b6fdbf99-4259-43bb-bbe7-f3c2a0168a87",
    "token_type": "bearer",
    "refresh_token": "eda15dc0-5b4a-4bc3-92c8-ffeeb63f7ded",
    "expires_in": 31169,
    "scope": "xx"
}

access_to_refresh:b6fdbf99-4259-43bb-bbe7-f3c2a0168a87 ->
	eda15dc0-5b4a-4bc3-92c8-ffeeb63f7ded
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

- 关闭 Spring Security 自带的 csrf 保护

```
@Override
public void configure(HttpSecurity http) throws Exception {
    /**
    * 1. 关闭 csrf，原因：
    *      - 提供的都是 restful 接口，并且通过外网调用必须在 header 中设置 token 信息，已经可以保证安全了，无需 csrf 防护；
    *      - 默认的 CsrfFilter 不支持 post 等方法，这样访问 post 方法时，需要提供 _csrf 的 token，在已经安全的情况下，没有必要。
    * 2. 可以设置部分接口无需认证，比如 /health，这样就可以通过 spring-boot-admin 来统一监控微服务。
    */
    http.csrf().disable()
            .exceptionHandling()
            .authenticationEntryPoint((request, response, authException) -> response.sendError(HttpServletResponse.SC_UNAUTHORIZED))
            .and()
            .authorizeRequests()    // "/health"接口无需认证，其余所有接口都需要认证
                .antMatchers("/health").permitAll()
                .anyRequest().authenticated()
            .and()
            .httpBasic();
}
```

完整代码参考： [GitHub](https://github.com/pggsnap/oauth)