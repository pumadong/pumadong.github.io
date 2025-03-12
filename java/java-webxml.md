---

layout: single
title: Java学习（Web.xml解析）
permalink: /java/java-webxml.html

classes: wide

author: Bob Dong

---

# 前言

Windows的IIS，是用UI界面进行站点的配置；Linux下面的几乎所有系统，都是使用配置文件来进行配置，Java容器（JBoss/Tomcat/Jetty/WebSphere/WebLogic等等）也不例外，它们使用一个部署在WEB-INFO目录下面的web.xml来作为站点配置文件。

本文参考互联网文章，学习并记录web.xml的加载顺序及配置详解。



web.xml加载顺序


应用服务器启动时web.xml的加载过程，和这些节点在xml文件中的前后顺序没有关系，不过有些应用服务器，比如WebSphere，就严格要求web.xml的节点顺序，否则部署不成功，所以最好还是按照web.xml的标准格式写，即：context-param --> listener --> filter --> servlet 。

或者根据IDE的提示，比如，如果顺序不对，IDE可能有如下提示：

The content of element type "web-app" must match "(icon?,display-
 name?,description?,distributable?,context-param*,filter*,filter-mapping*,listener*,servlet*,servlet-
 mapping*,session-config?,mime-mapping*,welcome-file-list?,error-page*,taglib*,resource-env-ref*,resource-
 ref*,security-constraint*,login-config?,security-role*,env-entry*,ejb-ref*,ejb-local-ref*)".

启动WEB项目的时候,应用服务器会去读它的配置文件web.xml，读两个节点：<listener></listener> 和 <context-param></context-param>   
紧接着，容器创建一个ServletContext(上下文)，这个WEB项目所有部分都将共享这个上下文
容器将<context-param></context-param>转化为键值对，并交给ServletContext
容器创建<listener></listener>中的类实例，即创建监听
在监听中会有contextInitialized(ServletContextEvent args)初始化方法，在这个方法中获得：
 ServletContext = ServletContextEvent.getServletContext();
 context-param的值 = ServletContext.getInitParameter("context-param的键"); 
得到这个context-param的值之后，就可以做一些操作了。注意,这个时候WEB项目还没有完全启动完成，这个动作会比所有的Servlet都要早。换句话说，这个时候，你对<context-param>中的键值做的操作，将在你的WEB项目完全启动之前被执行，如果想在项目启动之前就打开数据库，那么就可以在<context-param>中设置数据库的连接方式，在监听类中初始化数据库的连接，这个监听是自己写的一个类，除了初始化方法，它还有销毁方法，用于关闭应用前释放资源，比如说数据库连接的关闭。
对于某类配置节而言，与它们出现的顺序是有关的。

以 filter 为例，web.xml 中当然可以定义多个 filter，与 filter 相关的一个配置节是 filter-mapping，这里一定要注意，对于拥有相同 filter-name 的 filter 和 filter-mapping 配置节而言，filter-mapping 必须出现在 filter 之后，否则当解析到 filter-mapping 时，它所对应的 filter-name 还未定义。

web 容器启动时初始化每个 filter 时，是按照 filter 配置节出现的顺序来初始化的，当请求资源匹配多个 filter-mapping 时，filter 拦截资源是按照 filter-mapping 配置节出现的顺序来依次调用 doFilter() 方法的。 

servlet 同 filter 类似，此处不再赘述。

比如filter 需要用到 bean ，但加载顺序是： 先加载filter 后加载spring，则filter中初始化操作中的bean为null；所以，如果过滤器中要使用到 bean，可以将spring 的加载 改成 Listener的方式：

```
 <listener>
     <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
 </listener>
```

web.xml节点解析

<context-param />


用来设定web站点的环境参数

它包含两个子元素：<param-name></param-name> 用来指定参数的名称；<param-value></param-value> 用来设定参数值

在此设定的参数，可以在servlet中用 getServletContext().getInitParameter("my_param") 来取得

例子:

```
<context-param>
    <param-name>webAppRootKey</param-name>
    <param-value>privilege.root</param-value>
</context-param>
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>
        classpath*:/applicationContext*.xml
        classpath*:/cas-authority.xml
    </param-value>
</context-param>
<context-param>
    <param-name>log4jConfigLocation</param-name>
    <param-value>/WEB-INF/classes/log4j.xml</param-value>
</context-param>
<context-param>  
    <param-name>spring.profiles.default</param-name>  
    <param-value>dev</param-value>  
</context-param> 
```

### <listener />



用来设定Listener接口

它的主要子元素为 <listener-class></listener-class> ，用来定义Listener的类名称

例子:

```
 <listener>
     <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
 </listener>
```

<filter />


用来声明filter的相关设定

<filter-name></filter-name> 指定filter的名字

<filter-class></filter-class> 用来定义filter的类的名称

<init-param></init-param> 用来定义参数，它有两个子元素： <param-name></param-name> 用来指定参数的名称， <param-value></param-value> 用来设定参数值

与<filter></filter>一起使用的是

<filter-mapping></filter-mapping> 用来定义filter所对应的URL，包含两个子元素：

<filter-name></filter-name> 指定filter的名称

<url-pattern></url-pattern> 指定filter所对应的URL

 例子:

```
<!-- 解决中文乱码问题 -->
<filter>
    <filter-name>encodingFilter</filter-name>
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
    <init-param>
        <param-name>encoding</param-name>
        <param-value>UTF-8</param-value>
    </init-param>
    <init-param>
        <param-name>forceEncoding</param-name>
        <param-value>true</param-value>
    </init-param>
</filter>
<filter-mapping>
    <filter-name>encodingFilter</filter-name>
    <url-pattern>*.do</url-pattern>
</filter-mapping>
```

<servlet /> 


用来声明一个servlet的数据，主要有以下子元素：

<servlet-name></servlet-name> 指定servlet的名称

<servlet-class></servlet-class> 指定servlet的类名称

<jsp-file></jsp-file> 指定web站台中的某个JSP网页的完整路径

<init-param></init-param> 用来定义参数

与<servlet></servlet>一起使用的是

<servlet-mapping></servlet-mapping> 用来定义servlet所对应的URL，包含两个子元素：

<servlet-name></servlet-name> 指定servlet的名称

<url-pattern></url-pattern> 指定servlet所对应的URL

例子:

```
<servlet>
    <servlet-name>springmvc</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>springmvc</servlet-name>
    <url-pattern>*.do</url-pattern>
</servlet-mapping>

<!-- 用dubbo提供hessian服务 -->
<servlet>
    <servlet-name>dubbo</servlet-name>
    <servlet-class>com.alibaba.dubbo.remoting.http.servlet.DispatcherServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>dubbo</servlet-name>
    <url-pattern>/hessian/*</url-pattern>
</servlet-mapping>
```

基本节点


1、<description/> 是对站点的描述

例子：<description>传道、授业、解惑</description> 



2、<display-name/> 定义站点的名称

例子：<display-name>我的站点</display-name>



3、<icon> 

icon元素包含small-icon和large-icon两个子元素，用来指定web站点中小图标和大图标的路径。

<small-icon>/路径/smallicon.gif</small-icon>

small-icon元素应指向web站台中某个小图标的路径,大小为16 X 16 pixel，但是图象文件必须为GIF或JPEG格式,扩展名必须为：.gif或.jpg。

<large-icon>/路径/largeicon-jpg</large-icon>

large-icon元素应指向web站台中某个大图表路径,大小为32 X 32 pixel，但是图象文件必须为GIF或JPEG的格式,扩展名必须为： gif或jpg。

例子:

```
<icon> 
    <small-icon>/images/small.gif</small-icon> 
    <large-icon>/images/large.gir</large-icon> 
</icon>
```

4、 <distributable/> 是指定该站点是否可分布式处理



5、 <session-config/> 用来定义web站台中的session参数

包含一个子元素：

<session-timeout></session-timeout> 用来定义这个web站台所有session的有效期限，单位为 分钟



6、 <mime-mapping /> 定义某一个扩展名和某一个MIME Type做对应，它包含两个子元素：

<extension></extension> 扩展名的名称

<mime-type></mime-type> MIME格式

例子：

```
<mime-mapping>
    <extension>csv</extension>
    <mime-type>application/octet-stream</mime-type>
</mime-mapping>
```

7、 <error-page>

通过错误码来配置error-page

```
 <error-page>
     <error-code>404</error-code>
     <location>/message.jsp</location>
 </error-page>
```

通过异常类来配置error-page

```
<error-page>    
   <exception-type>java.lang.NullException</exception-type>    
   <location>/error.jsp</location>    
</error-page> 
```

8、 <welcome-file-list/>

```
<welcome-file-list>
    <welcome-file>index.html</welcome-file>
    <welcome-file>index.jsp</welcome-file>
</welcome-file-list>
```

9、 <resource-ref></resource-ref> 定义利用JNDI取得站台可利用的资源

有五个子元素：

<description></description> 资源说明

<rec-ref-name></rec-ref-name> 资源名称

<res-type></res-type> 资源种类

<res-auth></res-auth> 资源经由Application或Container来许可

<res-sharing-scope></res-sharing-scope> 资源是否可以共享，有Shareable和Unshareable两个值，默认为Shareable

比如，配置数据库连接池就可在此配置

```
<resource-ref>
    <description>JNDI JDBC DataSource of shop</description>
    <res-ref-name>jdbc/sample_db</res-ref-name>
    <res-type>javax.sql.DataSource</res-type>
    <res-auth>Container</res-auth>
</resource-ref>
```

WebApplicationInitializer


也不是说web.xml是100%需要的，如果使用了Spring，也可以用纯代码的方式来代替web.xml。

http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/WebApplicationInitializer.html

http://www.java-allandsundry.com/2014/09/spring-webapplicationinitializer-and.html

如果既配置了web.xml，又有一个实现了WebApplicationInitializer的类，则可能会报错呢，如下：

```
java.lang.IllegalStateException: Cannot initialize context because there is already a root application context present - check whether you have multiple ContextLoader* definitions in your web.xml!
	at org.springframework.web.context.ContextLoader.initWebApplicationContext(ContextLoader.java:277) ~[spring-web-4.1.1.RELEASE.jar:4.1.1.RELEASE]
	at org.springframework.web.context.ContextLoaderListener.contextInitialized(ContextLoaderListener.java:106) ~[spring-web-4.1.1.RELEASE.jar:4.1.1.RELEASE]
	at org.eclipse.jetty.server.handler.ContextHandler.callContextInitialized(ContextHandler.java:775) ~[jetty-server-8.1.10.v20130312.jar:8.1.10.v20130312]
	at org.eclipse.jetty.servlet.ServletContextHandler.callContextInitialized(ServletContextHandler.java:424) ~[jetty-servlet-8.1.10.v20130312.jar:8.1.10.v20130312]
	at org.eclipse.jetty.server.handler.ContextHandler.startContext(ContextHandler.java:767) ~[jetty-server-8.1.10.v20130312.jar:8.1.10.v20130312]
	at org.eclipse.jetty.servlet.ServletContextHandler.startContext(ServletContextHandler.java:249) ~[jetty-servlet-8.1.10.v20130312.jar:8.1.10.v20130312]
	at org.eclipse.jetty.webapp.WebAppContext.startContext(WebAppContext.java:1252) ~[jetty-webapp-8.1.10.v20130312.jar:8.1.10.v20130312]
	at org.eclipse.jetty.server.handler.ContextHandler.doStart(ContextHandler.java:710) ~[jetty-server-8.1.10.v20130312.jar:8.1.10.v20130312]
	at org.eclipse.jetty.webapp.WebAppContext.doStart(WebAppContext.java:494) ~[jetty-webapp-8.1.10.v20130312.jar:8.1.10.v20130312]
	at org.eclipse.jetty.util.component.AbstractLifeCycle.start(AbstractLifeCycle.java:64) [jetty-util-8.1.10.v20130312.jar:8.1.10.v20130312]
	at org.eclipse.jetty.server.handler.HandlerCollection.doStart(HandlerCollection.java:229) [jetty-server-8.1.10.v20130312.jar:8.1.10.v20130312]
	at org.eclipse.jetty.util.component.AbstractLifeCycle.start(AbstractLifeCycle.java:64) [jetty-util-8.1.10.v20130312.jar:8.1.10.v20130312]
	at org.eclipse.jetty.server.handler.HandlerWrapper.doStart(HandlerWrapper.java:95) [jetty-server-8.1.10.v20130312.jar:8.1.10.v20130312]
	at org.eclipse.jetty.server.Server.doStart(Server.java:280) [jetty-server-8.1.10.v20130312.jar:8.1.10.v20130312]
	at org.eclipse.jetty.util.component.AbstractLifeCycle.start(AbstractLifeCycle.java:64) [jetty-util-8.1.10.v20130312.jar:8.1.10.v20130312]
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:1.7.0_79]
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57) ~[na:1.7.0_79]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.7.0_79]
	at java.lang.reflect.Method.invoke(Method.java:606) ~[na:1.7.0_79]
```

推荐web.xml的配置方式，这种方式是最常用和通用的。

# 后记