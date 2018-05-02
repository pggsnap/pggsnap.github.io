---
layout:     post
title:      "Spring Cloud 系列之 Zuul 分析: 过滤"
date:       2018-05-02
author:     "pggsnap"
tags:
    - Spring Cloud
    - Zuul
---

以下分析基于 Spring Cloud 版本为 Edgware.SR3。

# 一、过滤器 ZuulServlet

查看下 ZuulServlet 的 service 方法源码：
```
public void service(ServletRequest servletRequest, ServletResponse servletResponse) throws ServletException, IOException {
    try {
        this.init((HttpServletRequest)servletRequest, (HttpServletResponse)servletResponse);
        RequestContext context = RequestContext.getCurrentContext();
        context.setZuulEngineRan();

        try {
            this.preRoute();
        } catch (ZuulException var12) {
            this.error(var12);
            this.postRoute();
            return;
        }

        try {
            this.route();
        } catch (ZuulException var13) {
            this.error(var13);
            this.postRoute();
            return;
        }

        try {
            this.postRoute();
        } catch (ZuulException var11) {
            this.error(var11);
        }
    } catch (Throwable var14) {
        this.error(new ZuulException(var14, 500, "UNHANDLED_EXCEPTION_" + var14.getClass().getName()));
    } finally {
        RequestContext.getCurrentContext().unset();
    }
}
```

请求基本流程：

1、preRoute -> route -> postRoute。

2、如果 preRoute 或者 route 过程出现 ZuulException 异常，则跳转到 error filter, 之后继续执行 postRoute。

3、如果 postRoute 出现异常，则执行 error filter。

# 二、过滤器类型
在 Spring Boot 中使用 @EnableZuulProxy 注解，会默认开启 endpoint 路由接口：/filters。设置 `management.security.enabled=false`，可以通过 `curl -X GET http://localhost:8080/filters` 获取路由信息：
```
{
    "pre":[
        {
            "class":"org.springframework.cloud.netflix.zuul.filters.pre.ServletDetectionFilter",
            "order":-3,
            "disabled":false,
            "static":true
        },
        {
            "class":"org.springframework.cloud.netflix.zuul.filters.pre.Servlet30WrapperFilter",
            "order":-2,
            "disabled":false,
            "static":true
        },
        {
            "class":"org.springframework.cloud.netflix.zuul.filters.pre.FormBodyWrapperFilter",
            "order":-1,
            "disabled":false,
            "static":true
        },
        {
            "class":"org.springframework.cloud.sleuth.instrument.zuul.TracePreZuulFilter",
            "order":0,
            "disabled":false,
            "static":true
        },
        {
            "class":"org.springframework.cloud.netflix.zuul.filters.pre.DebugFilter",
            "order":1,
            "disabled":false,
            "static":true
        },
        {
            "class":"org.springframework.cloud.netflix.zuul.filters.pre.PreDecorationFilter",
            "order":5,
            "disabled":false,
            "static":true
        }
    ],
    "route":[
        {
            "class":"org.springframework.cloud.netflix.zuul.filters.route.RibbonRoutingFilter",
            "order":10,
            "disabled":false,
            "static":true
        },
        {
            "class":"org.springframework.cloud.netflix.zuul.filters.route.SimpleHostRoutingFilter",
            "order":100,
            "disabled":false,
            "static":true
        },
        {
            "class":"org.springframework.cloud.netflix.zuul.filters.route.SendForwardFilter",
            "order":500,
            "disabled":false,
            "static":true
        }
    ],
    "error":[
        {
            "class":"org.springframework.cloud.netflix.zuul.filters.post.SendErrorFilter",
            "order":0,
            "disabled":false,
            "static":true
        }
    ],
    "post":[
        {
            "class":"org.springframework.cloud.sleuth.instrument.zuul.TracePostZuulFilter",
            "order":0,
            "disabled":false,
            "static":true
        },
        {
            "class":"org.springframework.cloud.netflix.zuul.filters.post.SendResponseFilter",
            "order":1000,
            "disabled":false,
            "static":true
        }
    ]
}
```

# 三、自定义过滤器

假设我们需要自定义 error filter, 用来替换 SendErrorFilter 并自定义错误返回，应该如何处理呢？

先看一下 SendErrorFilter 的代码：
```
public class SendErrorFilter extends ZuulFilter {

	protected static final String SEND_ERROR_FILTER_RAN = "sendErrorFilter.ran";

	@Override
	public String filterType() {
		return "error";
	}

	@Override
	public int filterOrder() {
		return 0;
	}

	@Override
	public boolean shouldFilter() {
		RequestContext ctx = RequestContext.getCurrentContext();
		// only forward to errorPath if it hasn't been forwarded to already
		return ctx.getThrowable() != null
				&& !ctx.getBoolean(SEND_ERROR_FILTER_RAN, false);
	}
}
```

由于同一类型的 filter 按照值由小到大的次序执行，因此为了替换 SendErrorFilter，自定义的 filterOrder 应该小于 0，并且需要设置 ctx.getBoolean(SEND_ERROR_FILTER_RAN, false) 为 true，那么 SendErrorFilter 就不会执行了。

现在实现了一个权限认证的 pre filter，如果验证不通过，则抛出异常，部分代码如下:
```
public class AccessFilter extends ZuulFilter {
    @Override
    public String filterType() {
        return "pre";
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        String authorization = request.getHeader("Authorization");
        if (authorization == null) {
            throw new OAuthException(HttpServletResponse.SC_UNAUTHORIZED, "Missing token.");
        }

        return null;
    }
}
```

其中自定义异常 OAuthException 继承自 RuntimeException，虽然抛出的是 OAuthException，但最终还是转化为 ZuulException，从而被捕获后执行 error filter。具体代码可以查看 FilterProcessor 类的 preRoute 方法。
```
public void preRoute() throws ZuulException {
    try {
        this.runFilters("pre");
    } catch (ZuulException var2) {
        throw var2;
    } catch (Throwable var3) {
        throw new ZuulException(var3, 500, "UNCAUGHT_EXCEPTION_IN_PRE_FILTER_" + var3.getClass().getName());
    }
}
```

从以上代码可见，如果你需要给用户展示一个 401 错误，有两种方法：

1、抛出 ZuulException(Throwable var1, 401, String errorCause) 异常；

2、抛出其他异常，但是会被封装为 ZuulException(Throwable var1, 500, String errorCause) 异常；需要自定义 error filter 解析 var1，获取 var1 异常中的其他信息。

如果采用方法 2，那么可以自定义 OAuthErrorFilter：
```
public class OAuthErrorFilter extends ZuulFilter {
    private String errorPath = "/error";

    @Override
    public String filterType() {
        return "error";
    }

    @Override
    public int filterOrder() {
        return -5;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() {
        try {
            RequestContext ctx = RequestContext.getCurrentContext();
            Throwable throwable = ctx.getThrowable();
            OAuthException oauthException;
            // throwable 为 ZuulException 类型，throw.getCause() 为 OAuthException 类型
            if (throwable.getCause() instanceof OAuthException) {
                oauthException = (OAuthException) throwable.getCause();
                log.warn(oauthException.getMessage(), oauthException);
                HttpServletRequest request = ctx.getRequest();
                RequestDispatcher dispatcher = request.getRequestDispatcher(this.errorPath);
                if (dispatcher != null) {
                    ctx.set("sendErrorFilter.ran", true);   // 这样就不会执行 SendErrorFilter 类中的 run 方法
                    if (!ctx.getResponse().isCommitted()) {
                        ctx.setResponseStatusCode(oauthException.getCode());
                        dispatcher.forward(request, ctx.getResponse());     // 执行自定义 /error 方法
                    }
                }
            }
        } catch (Exception var1) {
            ReflectionUtils.rethrowRuntimeException(var1);
        }

        return null;
    }
}
```
