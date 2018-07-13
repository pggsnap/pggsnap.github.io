---
layout:     post
title:      "Spring Cloud 升级 2.0 踩坑总结"
date:       2018-07-13
author:     "pggsnap"
tags:
    - Spring Cloud
    - Finchley
    - Spring Security
    - OAuth
---

# 升级 spring-security-oauth2 以及 spring-boot-starter-security

升级后出现异常 `There is no PasswordEncoder mapped for the id` 或者告警 `BCryptPasswordEncoder     : Encoded password does not look like BCrypt`。

原先代码如下：

```java
@Configuration
@EnableAuthorizationServer
public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {
	@Override
	public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
		clients.inMemory()
				.withClient("server")
				.scopes("xx")
				.secret("server")
				.authorizedGrantTypes("password", "refresh_token")
				.accessTokenValiditySeconds(43200)
				.refreshTokenValiditySeconds(2592000);
	}
}

@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
	@Autowired
	private MyUserDetailService myUserDetailService;

	@Bean
	public AuthenticationManager authenticationManager() throws Exception {
		return super.authenticationManager();
	}

	@Bean
	public PasswordEncoder passwordEncoder() {
		return new BCryptPasswordEncoder();
	}

	@Override
	protected void configure(AuthenticationManagerBuilder auth) throws Exception {
		auth.userDetailsService(myUserDetailService).passwordEncoder(passwordEncoder());
	}
}
```

Spring Security 升级 5.0 之后，不需要配置密码的加密方式，而是通过在密码前加前缀的方式表明加密方式；通过工厂类 PasswordEncoderFactories 来设置 passwordEncoder。

可以通过以下代码查看区别：
```java
public static void main(String[] args) {
    String pass = "123456";

    BCryptPasswordEncoder encode = new BCryptPasswordEncoder();
    String hashPass = encode.encode(pass);
    System.out.println(hashPass);       // $2a$10$M82w/e.Mi6S1bFoGAByYwOfcuDPjpBK9Pkywk7XM2vLxXwZk3AvzC

    PasswordEncoder passwordEncoder = PasswordEncoderFactories.createDelegatingPasswordEncoder();
    String hashPass2 = passwordEncoder.encode(pass);    // {bcrypt}$2a$10$wr1V8zCDSvnXNgZZkKSRIOIRZJLt885I/ZyoNf3RA0k1zVS3LbHWu
    System.out.println(hashPass2);
}
```

PasswordEncoderFactories 类的实现如下：
```java
public class PasswordEncoderFactories {
    public static PasswordEncoder createDelegatingPasswordEncoder() {
        String encodingId = "bcrypt";
        Map<String, PasswordEncoder> encoders = new HashMap();
        encoders.put(encodingId, new BCryptPasswordEncoder());
        encoders.put("ldap", new LdapShaPasswordEncoder());
        encoders.put("MD4", new Md4PasswordEncoder());
        encoders.put("MD5", new MessageDigestPasswordEncoder("MD5"));
        encoders.put("noop", NoOpPasswordEncoder.getInstance());
        encoders.put("pbkdf2", new Pbkdf2PasswordEncoder());
        encoders.put("scrypt", new SCryptPasswordEncoder());
        encoders.put("SHA-1", new MessageDigestPasswordEncoder("SHA-1"));
        encoders.put("SHA-256", new MessageDigestPasswordEncoder("SHA-256"));
        encoders.put("sha256", new StandardPasswordEncoder());
        return new DelegatingPasswordEncoder(encodingId, encoders);     // 默认为 bcrypt 加密
    }
}
```

因此，只需要将密码 password 修改为 {加密方式}password 即可。

1. 将存储在数据库中的用户密码加前缀。

2. 由于 spring-security-oauth2 也是基于 Spring Security，因此也需要修改 oauth2 中的 secret。

代码修改后如下：
```java
@Configuration
@EnableAuthorizationServer
public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {
	@Override
	public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
		clients.inMemory()
				.withClient("server")
				.scopes("xx")
				.secret("{bcrypt}$2a$10$FsY/rjYsU96BAG/jjCnzkeCN8mdmuXpXHkakVqjPREBrTajqalEoa")     // 修改 secret
				.authorizedGrantTypes("password", "refresh_token")
				.accessTokenValiditySeconds(43200)
				.refreshTokenValiditySeconds(2592000);
	}
}

@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
	@Autowired
	private MyUserDetailService myUserDetailService;

	@Bean
	public AuthenticationManager authenticationManager() throws Exception {
		return super.authenticationManager();
	}

	@Bean
	public PasswordEncoder passwordEncoder() {
		return PasswordEncoderFactories.createDelegatingPasswordEncoder();    // 采用新的加密格式
	}

	@Override
	protected void configure(AuthenticationManagerBuilder auth) throws Exception {
		auth.userDetailsService(myUserDetailService).passwordEncoder(passwordEncoder());
	}
}
```
