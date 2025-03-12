---

layout: single
title: 读一本书《Spring3.X企业应用开发实战》--SpringMVC
permalink: /java/spring3-mvc.html

classes: wide

author: Bob Dong

---

#  前言

本篇是《Spring3.X企业应用开发实战》，陈雄华 林开雄著，电子工业出版社，2012.2出版”的学习笔记的第三篇，关于SpringMVC。

Spring MVC 3.0和早期版本相比拥有了一个质的飞跃，全面支持REST风格的WEB编程、完全注解驱动、处理方法签名非常灵活、处理方法不依赖于Servlet API等。

由于Spring MVC框架在后头做了非常多的隐性工作，所以想深入掌握Spring MVC 3.0并非易事，本章我们在学习Spring MVC的各项功能时，还深入其内部了解其后台的运作机理，只有了解这些机理后，才能更好地使用这个当前最先进的MVC框架。

服务器启动时加载配置文件的顺序：web.xml  applicationContext.xml  springmvc-servlet.xml。



Spring MVC概述


Spring MVC通过一套MVC注解，让POJO成为处理请求的控制器，无须实现任何接口，同时，Spring MVC还支持REST风格的URL请求：注解驱动及REST风格的Spring MVC是Spring 3.0最出彩的功能之一。

此外，Spring MVC在数据绑定、视图解析、本地化处理及静态资源处理上都有不俗的表现。



DispatcherServlet


Spring MVC框架围绕DispatcherServlet这个核心展开，DispatcherServlet是Spring MVC的总导演、总策划，它负责截获请求并将其分派给相应的处理器处理。

在web.xml中配置DispatcherServlet，截获特定的URL请求：

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
```

Spring如何将上下文中的SpringMVC组件装配到DispatcherServlet中？



WebApplicationContext初始化后，此时Spring上下文中的Bean已经初始化完毕，开始执行DispatcherServlet的initStrategies()方法，代码如下：

```
protected void initStrategies(ApplicationContext context) {
 initMultipartResolver(); //1.初始化上传文件解析器
 initLocalResolver();//2.初始化本地化解析器
 initThemeResolver();//3.初始化主题解析器
 initHandlerMappings();//4.初始化处理器映射器
 initHandlerAdapters();//5.初始化处理器适配器
 initHandlerExceptionResolver();//6.初始化处理器异常解析器
 initRequestToViewNameTranslator();//7.初始化请求道试图名解析器
 initViewResolvers();//8.初始化试图解析器
}
```

该方法的工作是通过反射机制查找并装配Spring容器中显式自定义的组件Bean，如果找不到，则装配默认的组件实例，默认的组件实例在spring-webmvc-版本.RELEASE.jar包中，org/springframework/web/servlet路径下的DispatcherServlet.properties中间中定义。
其中文件上传没有默认的解析器，如果需要，自行配置，比如：

```
<!-- 配置对文件上传的支持 -->
<bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver" />
```

注解驱动的控制器


在POJO类定义处标注@Controller，再通过<context:component-scan/>扫描相应的类包，即可使POJO成为一个能处理HTTP请求的控制器。

@RequestMapping：不但支持标注的URL，还支持Ant风格（即?、*和**的字符）和带有{xxx}占位符的URL。带占位符的URL是Spring 3.0新增的功能，该功能在Spring MVC向REST目标挺进的发展过程中具有里程碑的意义。

通过@PathVariable可以将URL中的占位符参数绑定到控制器处理方法的入参中。



如何设置方法入参以绑定请求信息


其他参考：http://blog.csdn.net/kobejayandy/article/details/12690161

使用命令/表单对象绑定请求参数值：

这是最常用的，即入参就是一个POJO。Spring MVC会按照请求参数名和对象属性名进行匹配，自动为对象填充属性值，支持级联的属性名，例如：

@RequestMapping("/handleInsert)

public String handleInsert(User user,String operator)…


使用@RequestParam、@CookieValue、@RequestHeader，分别获取请求、Cookie、请求报文头的传入值，他们都有3个参数：

value：参数名；

required：是否必需，默认为true，表示请求中必须包含对应的参数名，如果不存在将抛出异常；

defaultValue：默认参数名，设置该参数时，自动将required设为false。极少情况需要使用该参数，也不推荐使用该参数



使用Servlet API对象作为入参：

public String handleInsert(HttpServletRequest request,HttpServletResponse response)...

另外Spring MVC在 org.springframework.web.context.request包中定义了若干个可代理Servlet原生API类的接口，如WebRequest和NativeWebRequest,他们也允许作为代理类的入参，通过这些代理类可访问请求对象的任何信息。


使用IO对象作为入参：

Servlet的ServletRequest拥有getInputStream()和getReader()方法，可以通过他们读取请求的信息。相应Servlet的ServletResponse拥有getOutputStream()和getWriter()方法，可以通过它们输出响应信息。

Spring MVC允许控制器的处理方法使用java.io.InputStream/java.io.Reader及java.io.OutputStream/java.io.Writer作为方法的入参，Spring MVC将获取ServletRequest的InputStream/Reader或ServletResponse的OutputStream/Writer，然后传递给控制器的处理方法。


使用其他类型的参数：

比如java.util.Locale、java.security.Principal，可以通过Servlet的HttpServletRequest的getLocale()及getUserPrincipal()得到相应的值。



HttpMessageConverter<T>进行消息对象转换


HttpMessageConverter<T>是Spring 3.0新添加的一个重要接口，它负责将请求信息转换为一个对象（类型为T)，将对象（类型为T）输出为响应信息。

DispatcherServlet默认已经安装了AnnotationMethodHandlerAdapter作为HandlerAdapter的组件实现类，HttpMessageConverter即由AnnotationMethodHandlerAdapter使用，

将请求信息转换为对象，或将对象转换为响应信息。

Spring为HttpMessageConverter<T>提供了众多的实现类，他们组成了一个功能强大、用途广泛的HttpMessageConverter<T>家族。

AnnotationMethodHandlerAdapter默认一级装配了如下的HttpMessageConverter：

StringHttpMessageConverter

ByteArrayHttpMessageConverter

SourceHttpMessageConverter

XmlAwareFormHttpMessageConverter

如果需要装配其他类型的HttpMessageConverter，可在Spring的Web容器上下文中自行定义一个AnnotationMethodHandlerAdapter，如果在Spring容器中显式定义了一个
AnnotationMethodHandlerAdapter，则Spring MVC将使用它覆盖默认的AnnotationMethodHandlerAdapter。



如何使用HttpMessageConverter<T>将请求信息转换并绑定到处理方法的入参中呢？
Spring MVC提供了两种途径：
1.使用@RequestBody/@ResponseBody对处理方法进行标注
2.使用HttpEntity<T>/ResponseEntity<T>作为处理方法的入参或返回值



关于HttpMessageConverter<T>，得出如下几条结论：

1.当控制器处理方法使用到@RequestBody/@ResponseBody 或 HttpEntity<T>/ResponseEntity<T>时，Spring MVC才使用注册的HttpMessageConverter对请求/响应消息进行处理

2.当控制器处理方法使用到@RequestBody/@ResponseBody 或 HttpEntity<T>/ResponseEntity<T>时，Spring首先根据请求头或响应头的Accept属性选择匹配的HttpMessageConverter，进而根据参数类型或反向类型的过滤得到匹配的HttpMessageConverter，如果找不到可用的HttpMessageConverter将报错

3.@RequestBody/@ResponseBody不需要成对出现，如果方法入参使用到@RequestBody，Spring MVC选择匹配的HttpMessageConverter将请求消息转换并绑定到该入参中。如果处理方法标注了

4.@ResponseBody，Spring MVC选择匹配的HttpMessageConverter将方法返回值转换并输出相应消息。

5.HttpEntity<T>/ResponseEntity<T>的功用和@RequestBody/@ResponseBody相似



RestTemplate


RestTemplate是Spring 3.0新增的模板类，在客户端程序中可使用该类调用Web服务端的服务，它支持REST风格的URL。此外，它项AnnotationMethodHandlerAdapter一样拥有一个httpMessageConverter的注册表，它默认注册了5个HttpMessageConverter。



spring-servlet.xml

spring-servlet.xml配置(WEB-INFO目录下)： 

```

<!-- 这是简单配置，代替bean节点那种显示加载bean的配置方式，可以自动加载必须得如下两个bean -->
<!-- <mvc:annotation-driven /> -->
<!-- 这是标准配置，可以解决ResponseBody中文乱码问题 -->
<bean  class="org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter">
<property name="messageConverters">
<list>
<bean
 class="org.springframework.http.converter.StringHttpMessageConverter">
 <property name="supportedMediaTypes">
  <list>
   <value>text/plain;charset=UTF-8</value>
  </list>
 </property>
 <property name="writeAcceptCharset" value="false"/>
</bean>
</list>
</property>
</bean> 
<bean class="org.springframework.web.servlet.mvc.annotation.DefaultAnnotationHandlerMapping" /> 
<!-- 装载拦截器  -->
<mvc:interceptors>
<!-- 更改语言环境时，一个'locale'的请求参数发送  -->
<bean class="org.springframework.web.servlet.i18n.LocaleChangeInterceptor" />  
<!-- 权限拦截  -->
<mvc:interceptor>
<mvc:mapping path="/**/*.do" />
<bean class="com.shopyp.authority.interceptor.AuthorityInterceptor"/>
</mvc:interceptor>  
</mvc:interceptors>
```

处理模型数据

对于MVC框架来说模型数据是最重要的，因为控制(C)是为了产生模型数据(M)，而视图(V)则是为了渲染模型数据。

如何将模型数据暴漏给视图是Spring MVC框架的一项重要工作，Spring MVC提供了多种途径输出模型数据，介绍如下：

1、ModelAndView：处理方法返回值为ModelAndView时，方法体即可通过该对象添加模型数据

2、@ModelAttribute：方法入参标注该注解后，入参的对象就会放到数据模型中

3、Map及Model：入参为org.springframework.ui.Model、org.springframework.ui.ModelMap或java.util.Map时，处理方法返回时，Map中的数据会自动添加到模型中

4、@SessionAttribute：将模型中的某个属性暂存到HttpSession中，以便多个请求之间可以共享这个属性



处理方法的数据绑定

我们知道Spring 会根据请求方法签名的不同，将请求信息中的信息以一定的方式转换并绑定到请求方法的入参中。

在请求消息到达真正调用处理方法的这一段时间内，Spring还完成了很多工作，包括数据转换、数据格式化及数据校验等。这一节使用较少，临时不去研究。



视图和视图解析器


请求处理方法执行完成后，最终返回一个ModelAndView对象。

对于那些返回String、View或ModelMap等类型的处理方法，Spring MVC也会在内部将它们装配成一个ModelAndView对象，它包含了视图逻辑名和模型对象的信息。

Spring MVC借助视图解析器（ViewResolver）得到最终的视图对象（View），这可能是我们常见的JSP视图，也可能是一个基于FreeMarker、Velocity模板技术的视图，还可能是PDF、Excel、XML、JSON等各种形式的视图。

对于最终究竟采取何种视图对象对模型对象进行渲染，Controller并不关心，Controller的工作重点聚集在生产模型数据的工作上，从而实现MVC的充分解耦。

FreeMarker配置：

```

 <!-- Freemarker配置，参考： http://www.cnblogs.com/hoojo/archive/2011/04/19/2020551.html-->
 <bean id="freemarkerConfig"
  class="org.springframework.web.servlet.view.freemarker.FreeMarkerConfigurer">
  <!-- 视图资源位置 -->
  <property name="templateLoaderPath" value="/WEB-INF/ftl/" />
  <property name="defaultEncoding" value="UTF-8" />
  <property name="freemarkerSettings">
   <props>
    <prop key="template_update_delay">0</prop><!-- 模板更新延时 -->
    <prop key="locale">zh_CN</prop>
    <prop key="default_encoding">UTF-8</prop>
    <prop key="output_encoding">UTF-8</prop>
    <prop key="template_exception_handler">rethrow</prop>
          <prop key="number_format">#.##</prop>
          <prop key="date_format">yyyy-MM-dd</prop>
          <prop key="time_format">HH:mm:ss</prop>
          <prop key="datetime_format">yyyy-MM-dd HH:mm:ss</prop>
   </props>
  </property>
  <!-- 全局变量部分 -->
  <property name="freemarkerVariables">
   <map>
    <entry key="BasePath" value="${web.basepath}" />
    <entry key="xml_escape" value-ref="fmXmlEscape" />
   </map>
  </property>
 </bean>
 <bean id="fmXmlEscape" class="freemarker.template.utility.XmlEscape" />
 <!-- 配置freeMarker视图解析器 -->
 <bean id="ftlviewResolver" class="org.springframework.web.servlet.view.freemarker.FreeMarkerViewResolver">
  <property name="viewClass" value="org.springframework.web.servlet.view.freemarker.FreeMarkerView" />
  <!-- 如果配置了这个节点，则视图必须是ftl，redirect等前缀都失效了 -->
  <!-- <property name="viewNames" value="*.ftl"/> -->
  <property name="contentType" value="text/html;charset=UTF-8" />
  <property name="cache" value="true" />
  <property name="prefix" value="" />
  <property name="suffix" value="" />
 </bean>
```

针对在实际开发过程中使用到的FreeMarker语法，JSTL标签等，可以逐渐熟悉和记录。
FreeMarker和Velocity是除JSP之外被使用最多的页面模板技术。页面模板编写好页面结构，模板页面中使用一些特殊的变量标识符绑定Java对象的动态数据。

FreeMarker是一个模板引擎，一个基于模板生产文本输出的通用工具，FreeMarker可以基于模板产生HTML、XML、JAVA源代码等多种类型的输出内容。虽然FreeMarker具有一些编程能力，单通常由Java程序准备数据，FreeMarker仅负责基于模板对模型数据进行渲染的工作。



关于Web开发


要想成为一名Web开发高手，不能仅满足于知道如何做，更要抛开现象探究本质。

笔者认为要成为一名Web开发高手，必须熟练了解如下内容：

1、每次请求和响应的背后究竟发生了哪些步骤，客户端服务器是如何通过HTTP请求报文进行交互的；

2、深刻掌握MIME类型的知识；

3、深刻掌握HTTP响应状态码的知识，如404、303究竟代表什么。



帖子：

1、http长连接：http://www.blogjava.net/xjacker/articles/334709.html

# 后记

