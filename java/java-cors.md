---

layout: single
title: Java学习（跨域）
permalink: /java/java-cors.html

classes: wide

author: Bob Dong

---

# 前言

什么是跨域


CORS全称：Cross-Origin Resource Sharing

在前后台分离的应用开发中，跨域是经常需要处理的场景。指的是访问不同域名的资源，对于静态资源的访问，比如CSS、GIF、Form请求，不存在跨域问题，一般说跨域问题，就是指的JavaScript的跨域问题以及Cookie的跨域使用问题（是使用，不是读取内容）。

跨域问题的根本原因是浏览器的安全限制，默认禁止跨域的动态资源请求。

参考文章：

http://www.cnblogs.com/rainman/archive/2011/02/20/1959325.html

AJAX POST&跨域 解决方案 - CORS

https://www.zhihu.com/question/26379635



Java服务器端跨域解决

一个Java应用，为了支持跨域，允许其他域名的JavaScript脚本访问本应用的资源，通常是在web.xml里面配置一个跨域处理Filter。

```
<!-- 允许跨域 -->
<filter>
    <filter-name>crossDomainFilter</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
    <init-param>
        <param-name>targetFilterLifecycle</param-name>
        <param-value>true</param-value>
    </init-param>
</filter>
<filter-mapping>
    <filter-name>crossDomainFilter</filter-name>
    <url-pattern>/*</url-pattern><!-- 配置允许跨域访问的的url-pattern -->
</filter-mapping>
```

跨域Filter的一般写法如下：

```

@Component("crossDomainFilter")
public class crossDomainFilter implements Filter {
 
    private static Logger errorLogger = LoggerFactory.getLogger(LogConstants.ERROR);
 
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    }
    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        try {
            HttpServletRequest httpRequest = (HttpServletRequest) request;
            HttpServletResponse httpResponse = (HttpServletResponse) response;
 
            // 跨域
            String origin = httpRequest.getHeader("Origin");
            if (origin == null) {
                httpResponse.addHeader("Access-Control-Allow-Origin", "*");
            } else {
                httpResponse.addHeader("Access-Control-Allow-Origin", origin);
            }
            httpResponse.addHeader("Access-Control-Allow-Headers", "Origin, x-requested-with, Content-Type, Accept,X-Cookie");
            httpResponse.addHeader("Access-Control-Allow-Credentials", "true");
            httpResponse.addHeader("Access-Control-Allow-Methods", "GET,POST,PUT,OPTIONS,DELETE");
            if ( httpRequest.getMethod().equals("OPTIONS") ) {
                httpResponse.setStatus(HttpServletResponse.SC_OK);
                return;
            }
            chain.doFilter(request, response);
        } catch (Exception e) {
            errorLogger.error("Exception in crossDomainFilter.doFilter", e);
            throw e;
        }
    }
    @Override
    public void destroy() {
    }
}
```

跨域解决办法原理


https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS



1、跨域时，浏览器会先发一次OPTIONS请求：OPTIONS 方法在跨域请求（CORS）中的应用

2、Access-Control-Allow-Origin与跨域：http://www.tuicool.com/articles/7FVnMz

3、Access-Control-Allow-Credentials于Cookie：http://zawa.iteye.com/blog/1868108 ，这在SSO中有用，调用方（a.com）把本应用（b.com）的cookie带给SSO，达到了登录认证的目的。



一个Cookie带不上的案例


一个真实项目中，跨域时Cookie没有带上，其中后端有：servletResponse.setHeader("Access-Control-Allow-Methods", "POST,GET,PUT,PATCH,DELETE");

前端也有：withCredentials: true 的配置。

解决问题的文章：https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS#Preflighted_requests

由于浏览器发送预检请求时，即OPTIONS，发现后端Access-Control-Allow-Methods没有OPTIONS，则后续的跨域请求就终止了，也就不会有后续的带Cookie的事情了。



前后台分离系统跨域时两种解决办法


场景：前端是a.com；后端是b.com，则解决跨域有如下两种场景：

1、配置好允许跨域相关配置，a.com请求b.com时，每次都通过跨域的配置，把cookie或者自定义Http头内容传给b.com，b.com拿到相关数据后进行身份验证；这种办法对于用户身份的cookie信息，实际是种在a.com下面的；这种方式需要前端处理登录，写Cookie等操作；

2、置好允许跨域相关配置，a.com请求b.com时，发现没有权限，则请求b.com的一个登录中转页面，b.com登录后，把用户身份信息种在b.com的cookie下面；这种方式，前端是纯展示层，不需要处理登录，写Cookie等操作。



总结


吃透了浏览器行为，和Http协议，就能对跨域有比较深刻的了解了。貌似这和前端更相关一些。实际的工作中，也是感觉前端对这些理解的更透彻，而后端的RD们更注重的是性能，比较不太关心这些技术。

# 后记

