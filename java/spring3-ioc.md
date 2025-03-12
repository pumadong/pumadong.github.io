---

layout: single
title: 读一本书《Spring3.X企业应用开发实战》--IoC和AOP
permalink: /java/spring3-ioc.html

classes: wide

author: Bob Dong

---

# 前言

本篇是“《Spring3.X企业应用开发实战》，陈雄华 林开雄著，电子工业出版社，2012.2出版”的学习笔记的第一篇，关于Spring最基础的IoC和AOP。


在日常的开发中，最近几年正在使用着Spring，过去使用过Spring.Net，从官方文档及互联网博客，看过很多Spring文章，出于各种原因，没有系统的进行Spring的学习，这次通过这本书系统的学习了Spring框架，很多知识贯穿起来，改变了一些错误理解，受益匪浅。

查看Spring源码的方法：

下载源码后，执行import-into-eclipse.sh(bat)，则会对源码建立Eclipse工程，Eclipse导入即可，执行这个批处理，需要JDK7及以上版本的支持。耐心一点，时间较长

在阅读Spring源码的过程中，会需要很多JDK反射及注解的知识，有过小小总结，如下：JDK框架简析--java.lang包中的基础类库、基础数据类型

另推荐一本书：《Spring Internals》Spring技术内幕，计文柯著，通过这本书，结合源代码，对于深入理解Spring架构和设计原理很有帮助。



使用Spring的好处到底在哪里？


你得先体会无Spring是什么滋味，才能知道Spring有何好处；

POJO编程，轻量级，低侵入；

面向接口编程，DI，解耦，降低业务对象替换的复杂性；

以提高开发效率为目标，简化第三方框架的使用方式；

灵活的基于核心 Spring 功能的 MVC 网页应用程序框架。开发者通过策略接口将拥有对该框架的高度控制，因而该框架将适应于多种呈现(View)技术，例如 JSP，FreeMarker，Velocity，Tiles，iText 以及 POI。值得注意的是，Spring 中间层可以轻易地结合于任何基于 MVC 框架的网页层，例如 Struts，WebWork，或 Tapestry；

他的作者说：

Spring是一个解决了许多在J2EE开发中常见的问题的强大框架。

Spring提供了管理业务对象的一致方法并且鼓励了注入对接口编程而不是对类编程的良好习惯。

Spring的架构基础是基于使用JavaBean属性的Inversion of Control容器。然而，这仅仅是完整图景中的一部分：Spring在使用IoC容器作为构建关注所有架构层的完整解决方案方面是独一无二的。 

Spring提供了唯一的数据访问抽象，包括简单和有效率的JDBC框架，极大的改进了效率并且减少了可能的错误。Spring的数据访问架构还集成了Hibernate和其他O/R mapping解决方案。 

Spring还提供了唯一的事务管理抽象，它能够在各种底层事务管理技术，例如JTA或者JDBC之上提供一个一致的编程模型。 

Spring提供了一个用标准Java语言编写的AOP框架，它给POJOs提供了声明式的事务管理和其他企业事务--如果你需要--还能实现你自己的aspects。这个框架足够强大，使得应用程序能够抛开EJB的复杂性，同时享受着和传统EJB相关的关键服务。 

Spring还提供了可以和总体的IoC容器集成的强大而灵活的MVC web框架。



IoC


定义


IoC(控制反转：Inverse of Control)是Spring容器的内核，AOP、声明式事务等功能在此基础上开花结果。

虽然IoC这个重要概念不容易理解，但它确实包含很多内涵，它涉及代码解耦、设计模式、代码优化等问题。

因为IoC概念的不容易理解，Martin Fowler提出了DI(依赖注入：Dependency Injection)的概念用来代替IoC，即让调用类对某一接口实现类的依赖关系由第三方（容器或协作类）注入，以移除调用类对某一接口实现类的依赖。

Spring通过一个配置文件描述Bean和Bean之间的依赖关系，利用Java语言的反射功能实例化Bean并建立Bean之间的依赖关系。 

Spring的IoC容器在完成这些底层工作的基础上，还提供了Bean实例缓存、生命周期管理、Bean实例代理、时间发布、资源装载等高级服务。



初始化


Bean工厂


概述


com.springframework.beans.factory.BeanFactory，是Spring框架最核心的接口，它提供了高级IoC的配置机制；使管理不同类型的Java对象成为可能；是Spring框架的基础设施，面向Spring本身。



初始化

```
ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
Resource res = resolver.getResource("classpath:com/baobaotao/beanFactory/benas.xml");
//ClassPathResource res = new ClassPathResource("com/baobaotao/beanFactory/benas.xml");
BeanFactory bf = new XmlBeanFactory(res);
System.out.println("init BeanFactory");
Car car = bf.getBean("car",Car.class);
System.out.println("car bean is ready for use!");
```

XmlBeanFactory通过Resource装载Spring配置信息并启动IoC容器，然后就可以通过getBean方法从IoC容器获取Bean了。

通过BeanFactory启动IoC容器时，不会初始化配置文件中定义的Bean，初始化动作发生在第一个调用时 。

对于SingleTon的Bean来说，BeanFactory会缓存Bean。



下面是一种更原始的编程式使用IoC容器的方法：

```

//以下演示一种更原始的载入和注册Bean过程，对我们了解IoC容器的工作原理很有帮助
//揭示了在IoC容器实现中的关键类，比如：Resource、DefaultListableBeanFactory、BeanDefinitionReader，之间的相互联系，相互协作
ClassPathResource res = new ClassPathResource("beans.xml");
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(res);
Car car = factory.getBean("car",Car.class);
System.out.println(car.toString());
```

**beans.xml的内容如下：**

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
    xsi:schemaLocation="http://www.springframework.org/schema/beans  http://www.springframework.org/schema/beans/spring-beans-4.0.xsd"
    >
	<bean id="car" class="cl.an.Car"></bean>
</beans>
```

ApplicationContext


概述


com.springframework.context.ApplicationContext，建立在BeanFactory基础上，提供了更多面向应用的功能，它提供了国际化支持和框架事件体系，更易于创建应用；面向使用Spring的开发者，几乎所有的应用场合我们都直接使用ApplicationContext而非底层的BeanFactory



初始化

```
ApplicationContext ctx = new ClassPathXmlApplicationContext(new String[]{"conf/beans1.xml","conf/beans2.xml"});
Car car = ctx.getBean("car",Car.class);
```

ApplicationContext的初始化和BeanFactory有一个重大的区别：
后者在初始化容器时，并未实例化Bean，直到第一次访问某个Bean时才实例目标Bean；而前者则在初始化应用上下文时就实例化所有单实例的Bean。因此ApplicationContext的初始化时间会比BeanFactory稍长一些，不过稍后的调用则没有“第一次惩罚”的问题；

另一个最大的区别，前者会利用Java反射机制自动识别出配置文件中定义的BeanPostProcessor、InstantiationAwareBeanPostPrcecssor和BeanFactoryPostProcessor，并自动将他们注册到应用上下文中；而后者需要在代码中通过手工调用addBeanPostProcessor方法进行注册。这也是为什么在应用开发时，我们普遍使用ApplicationContext而很少使用BeanFactory的原因之一。

可以在beans的属性中定义default-lazy-init="true"，达到延迟初始化的目的，这不能保证所有的bean都延迟初始化，因为有的bean可能被依赖导致初始化。不推荐延迟初始化。



WebApplicationContext


概述


WebApplicationContext是专门为Web应用准备的，它允许从相对于Web根目录的路径中装载配置文件完成初始化工作。

从WebApplicationContext中可以获得ServletContext的引用，整个Web应用上下文对象将作为属性放置到ServletContext中，以便Web应用环境可以访问Spring应用上下文。



初始化


WebApplicationContext的初始化方式和BeanFactory、ApplicationContext有所区别，因为WebApplicationContext需要ServletContext实例，也即是说它必须在Web容器的前提下才能完成启动的工作。有过Web开发经验的读者都知道可以在web.xml中配置自启动的Servlet或定义Web容器监听器，借助这两者中的任何一个，我们就可以完成启动Spring Web应用上下文的工作。

所有版本的Web容器都可以定义自启动的Servlet，但只有Servlet2.3及以上版本的Web容器才支持Web容器监听器。有些即使支持Servlet2.3的Web服务器，但也不能再Servlet初始化之前启动Web监听器，比如Weblogic 8.1,Websphere 5.x,Oracle OC4J 9.0。

Web.xml里面的配置节点如下：

```
<!--指定配置文件-->
<context-param>
  <param-name>contextConfigLocation</param-name>
  <param-value>
   classpath*:/applicationContext.xml
  </param-value>
</context-param>
<!--声明Web容器监听器-->
<listener>
  <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

log4j


需要一种日志框架，我们使用Log4J，在类路径下，提供Log4J配置文件log4j.xml，这样启动Spring容器才不会报错。

对于WebApplicationContext，可以将Log4J配置文件放置在WEB-INF/classes下，这时Log4J引擎即可顺利启动。如果Log4J配置文件放置在其他位置，用户还必须在web.xml指定Log4J配置文件位置。

Spring为启动Log4J引擎提供了两个类似于启动WebApplicationContext的实现类：Log4jConfigServlet和Log4jConfigListener，不管采用哪种方式都必须保证能在在装载Spring配置文件之前先装载Log4J配置文件。

Web.xml里面的配置节点如下：

```
<!--指定Log4J配置文件位置-->
<context-param>
  <param-name>log4jConfigLocation</param-name>
  <param-value>/WEB-INF/classes/log4j.xml</param-value>
</context-param>
<!--声明Log4J监听器-->
<listener>
  <listener-class>org.springframework.web.util.Log4jConfigListener</listener-class>
</listener>
```

对于maven java application项目，log4j.xml直接放在src/main/resource下面即可，否则会报红色警告：

```
二月 09, 2015 10:57:38 下午 org.springframework.context.support.ClassPathXmlApplicationContext prepareRefresh
信息: Refreshing org.springframework.context.support.ClassPathXmlApplicationContext@a6af6e: startup date [Mon Feb 09 22:57:38 CST 2015]; root of context hierarchy
二月 09, 2015 10:57:38 下午 org.springframework.beans.factory.xml.XmlBeanDefinitionReader loadBeanDefinitions
信息: Loading XML bean definitions from class path resource [applicationContext.xml]
```

#### 使用外部属性文件

```
<!-- 定义受环境影响易变的变量 -->
<bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
<property name="systemPropertiesModeName" value="SYSTEM_PROPERTIES_MODE_OVERRIDE" />
<property name="ignoreResourceNotFound" value="true" />
<property name="locations">
 <list>
  <!-- 标准配置 -->
  <value>classpath*:/config.properties</value>  
  <!-- 本地开发环境配置 -->
  <value>file:/d:/conf/*.properties</value>
  <!-- 服务器生产环境配置 -->
  <value>file:/etc/conf/*.properties</value>
 </list>
</property>
</bean>
```

在基于xml的配置方式中，通过${cas.server.url}的表达式即可访问配置信息；在基于注解和基于Java类配置的Bean中，可以通过@Value("${cas.server.url}")的注解形式访问配置信息。#是引用Bean的属性值。



总结


BeanFactory、ApplicationContext和WebApplicationContext是Spring框架三个最核心的接口，框架中其他大部分的类都围绕它们展开、为它们提供支持和服务。

在这些支持类中，Resource是一个不可忽视的重要接口，框架通过Resource实现了和具体资源的解耦，不论它们位于何种介质中，都可以通过相同的实例返回。

与Resource配合的另一个接口是ResourceLoader，ResourceLoader采用了策略模式，可以通过传入资源的信息，自动选择适合的底层资源实现类，为生产对资源的引用提供了极大的便利。

在一些非Web项目中，在入口函数（main）中，会显示的初始化Spring容器，比如：ApplicationContext instance = new ClassPathXmlApplicationContext(new String[]{"applicationContext.xml"})，这和Web项目中通过Listener（web.xml）初始化Spring容器，效果是一样的。一个是手工加载，一个是通过web容器加载，对于Spring容器的初始化，效果是一样的。



Bean生命周期和作用域


Spring为Bean提供了细致周全的生命周期过程，通过实现特定的接口或通过<bean>属性设置，都可以对Bean的生命周期过程施加影响，Bean的生命周期不但和其实现的接口相关，还与Bean的作用范围有关。为了让Bean绑定在Spring框架上，我们推荐使用配置方式而非接口方式进行Bean生命周期的控制。

在实际的开发过程中，我们很少控制Bean生命周期，而是把这个工作交给Spring，采用默认的方式。

Bean的作用域：singleton,prototype,request,session,globalSession，默认是singleton。



配置方式


基于Xml配置方式中，配置文件的3种格式：完整配置格式、简化配置方式、使用p命名空间 。

基于注解配置方式中，使用到的注解符号：@Compoment,@Repository,@Service,@Controller,@Autowired(@Resource,@Inject),@Qualifier,@Scope,@PostConstruct,@PreDestroy  。

基于Java类配置方式中，使用到的注解符号：@Configuration,@Bean 。

Bean不同配置方式比较，总结如下：

| 配置方式         | 基于XML配置                                                  | 基于注解配置                                                 | 基于Java类配置                                               |
| ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Bean定义         | 在XML文件中，通过<bean>元素定义Bean。如：<bean class="com.bbt.UserDao"/> | 在Bean实现类处通过标注@Component或衍生类（@Repository,@Service,@Controller）定义Bean | 在标注了Configuration的Java类中，通过在类方法上标注@Bean定义一个Bean。方法必须提供Bean的实例化逻辑。 |
| Bean名称         | 通过<bean>的id或name属性定义，如：<bean id=""userDao"" class=com.bbt.UserDao/>，默认名称为：com.bbt.UserDao#0 | 通过注解的Value属性定义，如@Component("userDao")。默认名称为小写字母打头的类名（不带包名）：userDao | 通过@Bean的name属性定义，如 @Bean("userDao")，默认名称为方法名。 |
| Bean注入         | 通过<property>子元素或通过p命名空间的动态属性，如p:userDao-ref="userDao"进行注入 | 通过在成员变量或方法入参处标注@Autowired，按类型匹配自动注入。还可以配合使用@Qualifier按名称匹配方式呼入 | 比较灵活，可以通过在方法除通过@Autowired使方法入参绑定Bean，然后在方法中通过代码进行注入，还可通过调研配置类的@Bean方法进行注入。 |
| Bean生命过程方法 | 通过<bena>的init-method和destroy-method属性指定Bean实现类的方法名最多只能指定一个初始化方法和一个销毁方法。 | 通过在目标方法上标注@PostConstruct和@PreDestroy注解指定初始化或销毁方法，可以定义任意多个 | 通过@Bean的initMethod或destroyMethod指定一个初始化或销毁方法；对于初始化方法来说，可以直接在方法内部通过代码的方式灵活定义初始化逻辑。 |
| Bean作用范围     | 通过<bean>的scope属性指定，如：<bean class="com.bbt.UserDao" scope="prototype"> | 通过在类定义处标注@Scope指定，如：@Scope("prototype")        | 通过在Bean方法定义处标注@Scope指定                           |
| Bean延迟初始化   | 通过<bean>的lazy-init属性绑定，默认为default，继承于<beans>的default-lazy-init设置，该值默认为false | 通过在类定义处标注@Lazy指定，如@Lazy(true)                   | 通过在类定义处标注@Lazy指定                                  |
| 适合场景         | 1.Bean实现类来源于第三方类库，如DataSource,JdbcTemplate等，因无法在类中标注注解，通过XML配置方式较好； 2.命名空间的配置，如aop,context等，只能采用基于XML的配置 | Bean的实现类是当前项目开发的，可以直接在Java类中使用基于注解的配置 | 基于Java类配置的优势在于可以通过代码方式控制Bean初始化的整体逻辑。所以如果实例化Bean的逻辑比较复杂，则比较适合用基于Java类配置的方式 |

一般采用XML配置DataSource，SessionFactory等资源Bean，在XML中利用aop，context命名空间进行相关主题的配置，其他的自己项目中开发的Bean，都通过基于注解配置的方式进行配置，即整个项目采用“基于XML+基于注解”的配置方式，很少采用基于Java类的配置方式。



通用知识点


1.Xml有5个特殊符号：<>&"'，转义字符分别为：<  >  &  "  '  ，也可以用<![CDATA[内容]]>的方式。

2.资源类型的地址前缀：class: class*:  file:  http://  ftp://

3.JavaBean规范规定：变量的前两个字母要么全部大写，要么全部小写

4.默认构造函数是不带参的构造函数。Java语言规定如果类中没有定义任何构造函数，则JVM自动为其生成一个默认的构造函数。反之，如果类中显式定义了构造函数，则JVM不会为其生成默认的构造函数。所以假设Car类中显式定义了一个带参的构造函数，如public Car(String brand)，则需要同时提供一个默认构造函数public Car()，否则使用属性注入时将抛出异常。



AOP


AOP，Aspect Oriented Programming，面向切面编程

AOP的出现，是作为OOP的有益补充；AOP的应用场合是受限的，它一般只适合于那些具有横切逻辑的应用场合：如性能监测、访问控制、事务管理、日志记录。

OOP是通过纵向继承的机制，达到代码重用的目的；AOP通过横向抽取机制，把分散在各个业务逻辑中的相同代码，抽取到一个独立的模块中，还业务逻辑类一个清新的世界；把这些横切性的逻辑独立出来很容易，但如何将这些横切逻辑融合到业务逻辑中完成和原来一样的业务操作，才是事情的关键，这也正是AOP要解决的主要问题。

"AOP术语：连接点（Joinpoint）、切点（Pointcut）、增强（Advice）、目标对象（Target）、引介（Introduction）、织入（Weaving）、代理（Proxy）、切面（Aspect）；
AOP织入方式：编译器织入、类装载期织入、动态代理织入，Spring采用动态代理织入，AspectJ采用前两种织入方式；"

Spring采用了两种代理机制：基于JDK的动态代理和基于CGLib的动态代理；前者创建的代理对象性能差，后者创建对象花费时间长，都大约是10倍的差距，所以对于单例的代理对象或者具有实例池的代理对象，比较适合CGLib，反之适合JDK。

关于AOP技术的实现技术步骤，先不进行深入研究，其中JDK5.0开始的注解技术，是对开发效率有明显改进的，可以深入了解下；实际所有基于注解的配置方式全部来源于JDK5.0注解技术的支持。

四种切面类型：@AspectJ、<aop:aspect>、Advisor、<aop:advisor>，JDK5.0以后全部采用@AspectJ方式吧。这四种切面定义方式，其底层实现实际是相同的，表象不同，本质归一。

1、@AspectJ使用JDK5.0注解和正规的AspectJ的切点表达式语言描述切面，由于Spring只支持方法的连接点，所以Spring仅支持部分AspectJ的切点语言，这种方式是被Spring推荐的，需要在xml中配置的内容最少，演示：http://blog.csdn.net/puma_dong/article/details/20863953#t28

2、如果项目只能使用低版本的JDK，则可以考虑使用<aop:aspect>，这是基于Schema的配置方式

3、如果正在升级一个基于低版本Spring AOP开发的项目，则可以考虑使用<aop:advisor>复用已经存在的Advice类

4、基于Advisor类的方式，其实也是蛮自动的，在xml中配置一个Advisor节点，开启自动创建代理就好了，比如：http://blog.csdn.net/puma_dong/article/details/20863953#t27


<!--开启注解 -->
 <context:annotation-config />
参考：http://blog.sina.com.cn/s/blog_872758480100wtfh.html
 <!-- 开启自动切面代理 -->
 <aop:aspectj-autoproxy />


声明自动为spring容器中那些配置@aspectJ切面的bean创建代理，织入切面。当然，spring在内部依旧采用AnnotationAwareAspectJAutoProxyCreator进行自动代理的创建工作，但具体实现的细节已经被<aop:aspectj-autoproxy />隐藏起来了


<aop:aspectj-autoproxy />有一个proxy-target-class属性，默认为false，表示使用jdk动态代理织入增强，当配为<aop:aspectj-autoproxy  proxy-target-class="true"/>时，表示使用CGLib动态代理技术织入增强。不过即使proxy-target-class设置为false，如果目标类没有声明接口，则spring将自动使用CGLib动态代理；关于这点，很多资料这么说，但我实际的测试是，如果没有接口，只能用cglib，否则会报异常“Exception in thread "main" org.springframework.beans.factory.BeanNotOfRequiredTypeException: Bean named 'person' must be of type [cl.an.Person], but was actually of type [cl.an.$Proxy9]”。



演示JDK动态代理实现AOP的基本原理

```
import java.lang.reflect.*;
class AOPTest {
	public static void main(String[] args) throws Exception {
		HelloInterface hello = BeanFactory.getBean("HelloImpl",HelloInterface.class);
		hello.setInfo("zhangsan","zhangsan@163.com");
	}
}
 
interface HelloInterface {
	public String setInfo(String name,String email);
}
 
class HelloImpl implements HelloInterface {
	private String name;
	private String email;
	@Override
	public String setInfo(String name,String email) {
		this.name = name;
		this.email = email;
		System.out.println("\n\n===>setInfo函数内部输出此行...");
		return "OK";
	}
}
 
class AOPHandler implements InvocationHandler {
	private Object target;
	public AOPHandler(Object target) {
		this.target = target;
	}
	public void println(String str,Object... args) {
		System.out.println(str);
		if(args == null) {
			System.out.println("\t\t\t 未传入任何值...");
		} else {
			for(Object obj:args) {
				System.out.println("\t\t\t" + obj);
			}
		}
	}
	@Override
	public Object invoke(Object proxyed,Method method,Object[] args)
	throws IllegalArgumentException,IllegalAccessException,InvocationTargetException 
	{
		//以下定义调用之前执行的操作
		System.out.println("\n\n===>调用方法名: " + method.getName());
		Class<?>[] variables = method.getParameterTypes();
		System.out.println("\n\t参数类型列表：\n");
		for(Class<?> typevariable : variables) {
			System.out.println("\t\t\t" + typevariable.getName());
		}
		println("\n\n\t传入参数值为：",args);
 
		//以下开始执行代理方法	
		System.out.println("\n\n开始执行method.invoke...调用代理方法...");
		Object result = method.invoke(target,args);
		
		//以下定义调用之后执行的操作
		println("\n\n\t返回的参数为：",result);
		println("\n\n\t返回值类型为：",method.getReturnType());
		return result;
	}
}
 
class BeanFactory {
	public static Object getBean(String className) throws InstantiationException,IllegalAccessException,ClassNotFoundException {
		Object obj = Class.forName(className).newInstance();
		InvocationHandler handler = new AOPHandler(obj);//定义过滤器
		return Proxy.newProxyInstance(obj.getClass().getClassLoader(),obj.getClass().getInterfaces(),handler);
	}
	@SuppressWarnings("unchecked")
	public static<T> T getBean(String className,Class<T> c) throws InstantiationException,IllegalAccessException,ClassNotFoundException {
		return (T)getBean(className);
	}
}
```

其他参考方案：http://javatar.iteye.com/blog/814426/



演示Cglib动态代理实现AOP的基本原理


对于没有通过接口定义业务方法的类，如何动态创建代理实例呢？JDK的代理技术显然已经黔驴技穷，CGLib作为一个替代者，填补了这个空缺。

CGLib采用非常底层的字节码技术，可以为一个类创建子类，并在子类中采用方法拦截的技术拦截所有父类的方法调用，并顺势织入横切逻辑。

以下会有代码演示，关于代码演示，最简单的就是创建一个maven项目，依赖cglib后，会自动找出相关依赖。

本例采用较笨的办法，先把cglib需要的jar下载下来cglib-3.1.jar和asm-4.2.jar，然后在记事本编写TestForumService.java，之后使用javac编译，java运行。

javac -classpath cglib-3.1.jar TestForumService.java

java -classpath .;cglib-3.1.jar;asm-4.2.jar TestForumService，对于windows使用;连接，对于linux，使用:连接。

```
import java.lang.reflect.Method;
import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;
 
public class TestForumService
{
	public static void main(String[] args) throws Exception {
		CglibProxy proxy = new CglibProxy();
		
		ForumServiceImpl forumService = (ForumServiceImpl)proxy.getProxy(ForumServiceImpl.class);
		forumService.removeTopic(100);
	}
}
 
class CglibProxy implements MethodInterceptor
{
	private Enhancer enhancer = new Enhancer();
 
	public Object getProxy(Class<?> clazz) {
		enhancer.setSuperclass(clazz);
		enhancer.setCallback(this);
		return enhancer.create();	//通过字节码技术动态创建子类实例
	}
	//拦截父类所有方法的调用
	public Object intercept(Object obj,Method method,Object[] args,MethodProxy proxy)
		throws Throwable {
		PerformanceMonitor.begin(obj.getClass().getName() + "." + method.getName());
		Object result = proxy.invokeSuper(obj, args);
		PerformanceMonitor.end();
		return result;
	}
}
 
//将要被注入切面功能的类
class ForumServiceImpl
{
	public void removeTopic(int topicId) throws Exception {
		System.out.println("模拟删除Topic记录：" + topicId);
		//Thread.sleep(20);
	}
}
 
//性能监视类（切面类）
class PerformanceMonitor
{
	private static ThreadLocal<MethodPerformance> performanceRecord = 
		new ThreadLocal<MethodPerformance>();
	
	public static void begin(String method) {
		System.out.println("begin monitor...");
		MethodPerformance mp = new MethodPerformance(method);
		performanceRecord.set(mp);
	}
 
	public static void end() {
		System.out.println("end monitor...");
		MethodPerformance mp = performanceRecord.get();
 
		mp.printPerformance();
	}
}
 
class MethodPerformance
{
	private long begin,end;
	private String serviceMethod;
	
	public MethodPerformance(String serviceMethod) {
		this.serviceMethod = serviceMethod;
		this.begin = System.currentTimeMillis();
	}
 
	public void printPerformance() {
		end = System.currentTimeMillis();
		long elapse = end - begin;
		System.out.println(serviceMethod + "花费" + elapse + "毫秒。");
	}
}
```

演示Spring项目的自定义注解

这个例子演示的是，SpringAOP通过自动代理（BeanPostProcessor）和切面（Advisor）来完成环绕增强（通过实现AOP联盟定义的MethodInterceptor接口）的过程，这个过程中，也会读取和分析注解，所以也是一个注解解析器，java代码如下：

```
package cl.an;
 
import java.lang.annotation.*;
 
import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import org.springframework.stereotype.Service;
import org.springframework.stereotype.Component;
 
public class MainTest {	
	public static void main(String[] args) {
		@SuppressWarnings("resource")
		ApplicationContext ctx = new ClassPathXmlApplicationContext(new String[]{"applicationContext.xml"}); 
		//测试IOC
		MainUtil mainUtil = ctx.getBean("mainUtil",MainUtil.class);
		if(mainUtil == null) {
			System.out.println("null");
		} else {
			System.out.println(mainUtil.getString());
		}
		//测试自定义注解
		Person person = ctx.getBean("person",Person.class);
		System.out.println(person.say());
	}
}
 
@Service
class MainUtil {
 
	@Autowired
	private MainConfigUtil configUtil;
	
	public String getString() {
		
		return "MainUtil." + configUtil.getString();
	}
	
}
 
@Service
class MainConfigUtil {
	public String getString() {
		return "MainConfigUtil";
	}
}
 
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@interface AnnotationTest {
	String value();
}
 
@Component("person")
class Person {
	@AnnotationTest("Annotation's Content")
	public String say(){
		return "I'm OK";
	}
}
 
@Component("annotationTestAdvice")
class AnnotationTestAdvice implements MethodInterceptor {
	public Object invoke(MethodInvocation invocation) throws Throwable {
		if(invocation.getMethod().isAnnotationPresent(AnnotationTest.class)){
			String content = null;
			Annotation annotation = invocation.getMethod().getAnnotation(AnnotationTest.class);
			if(annotation!=null){
				content = ((AnnotationTest)annotation).value();
				System.out.println("获取到注解的内容：" + content);
			}
			
			System.out.println("方法调用之前要进行的工作");
			Object o = invocation.proceed();
			System.out.println("方法调用之后要进行的工作");
			
			return o;
			
		}else{
			return invocation.proceed();
		}
	}
}
```

applicationContext.xml

```

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
    xmlns:jee="http://www.springframework.org/schema/jee"
    xmlns:tx="http://www.springframework.org/schema/tx" 
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:task="http://www.springframework.org/schema/task" 
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:p="http://www.springframework.org/schema/p"
    xmlns:cache="http://www.springframework.org/schema/cache"
    xmlns:c="http://www.springframework.org/schema/c"
    xmlns:util="http://www.springframework.org/schema/util"
    xsi:schemaLocation="http://www.springframework.org/schema/beans  http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
    http://www.springframework.org/schema/cache http://www.springframework.org/schema/cache/spring-cache.xsd
    http://www.springframework.org/schema/tx  http://www.springframework.org/schema/tx/spring-tx-4.0.xsd
    http://www.springframework.org/schema/jee  http://www.springframework.org/schema/jee/spring-jee-4.0.xsd
    http://www.springframework.org/schema/context  http://www.springframework.org/schema/context/spring-context-4.0.xsd
    http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-4.0.xsd
    http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util-4.0.xsd"
    default-lazy-init="true">
 
	<description>Spring公共配置</description>
 
	<!--开启注解 -->
	<context:annotation-config />
	
	<!-- 开启自动切面代理(AnnotationAwareAspectJAutoProxyCreator) -->
	<!-- true:代表使用cglib进行动态代理 -->
	<aop:aspectj-autoproxy proxy-target-class="true" />
	<!-- 开启自动切面代理(DefaultAdvisorAutoProxyCreator) -->
	<!-- <bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"/> -->
	
	<context:component-scan base-package="cl.an">
		<context:exclude-filter type="annotation" expression="org.springframework.stereotype.Controller" />
	</context:component-scan>
 
	<bean id="nameMatchMethodPointcutAdvisor" class="org.springframework.aop.support.NameMatchMethodPointcutAdvisor">    
    <property name="advice" ref="annotationTestAdvice"/>    
    <property name="mappedNames">    
        <list>    
            <value>say</value>    
        </list>    
    </property>
	</bean> 
</beans>
```

pom.xml

```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
 
  <groupId>cl</groupId>
  <artifactId>an</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <packaging>jar</packaging>
 
  <name>an</name>
  <url>http://maven.apache.org</url>
 
	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<spring.version>4.0.6.RELEASE</spring.version>
	</properties>
  
	<dependencies>
		<!-- logging -->
		<dependency>
			<groupId>log4j</groupId>
			<artifactId>log4j</artifactId>
			<version>1.2.16</version>
		</dependency>
		<dependency>
			<groupId>org.slf4j</groupId>
			<artifactId>slf4j-api</artifactId>
			<version>1.6.1</version>
		</dependency>
		<dependency>
			<groupId>org.slf4j</groupId>
			<artifactId>slf4j-log4j12</artifactId>
			<version>1.6.1</version>
		</dependency>
		<!-- Spring framework -->
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-core</artifactId>
			<version>${spring.version}</version>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-beans</artifactId>
			<version>${spring.version}</version>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-context</artifactId>
			<version>${spring.version}</version>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-context-support</artifactId>
			<version>${spring.version}</version>
		</dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-aspects</artifactId>
            <version>${spring.version}</version>
        </dependency>
	 </dependencies>   
</project>
```

**还可以只通过注解方式实现注解：**

```
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
    <version>1.8.7</version>
</dependency>
```

```
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface MonitorLog {
    public String MonitorKey() default "";
    public long TimeOut() default 200;//单位毫秒
}
```

```
@Aspect
public class MonitorLogAnnotationProcessor {
 
 
    @Around("execution(* *(..)) && @annotation(log)")
    public Object aroundMethod(ProceedingJoinPoint pjd ,MonitorLog log) {
        Object result = null;
        String monitorKey =  log.MonitorKey();
        long timeout = log.TimeOut();
        String e_monitorKey="";
        String monitorKey_timeout = "";
        if(monitorKey==null || "".equals(monitorKey)){
            MethodSignature signature = (MethodSignature) pjd.getSignature();
            Method method = signature.getMethod();
            String mName = method.getName();
            Class<?> cls =  method.getDeclaringClass() ;
            String cName = cls.getName() ;
            monitorKey= cName+"."+mName ;
        }
        e_monitorKey= monitorKey+".error" ;
        monitorKey_timeout = monitorKey+".timeout" ;
        long start = System.currentTimeMillis();
        try {
            result = pjd.proceed();
        } catch (Throwable e) {//todo 是否捕获所有异常
            JMonitor.add(e_monitorKey);
        }finally {
            long elapseTime = System.currentTimeMillis() - start;
            if(elapseTime > timeout) {
                JMonitor.add(monitorKey_timeout);
            }
            JMonitor.add(monitorKey, elapseTime);
        }
        return result;
    }
 
}
```

### SpringAOP实现方式从低级到高级



这个例子通过一个“前置增强”的逐步简化步骤，演示SpringAOP如何从最低级的实现，到最高级（最简化、最自动）的实现的进化过程。



#### 纯粹代码方式（ProxyFactory）

```
package cl.an.advice;
 
import java.lang.reflect.Method;
import org.springframework.aop.BeforeAdvice;
import org.springframework.aop.MethodBeforeAdvice;
import org.springframework.aop.framework.ProxyFactory;
 
public class TestBeforeAdvice {
	public static void main(String[] args) {
		Waiter target = new NaiveWaiter();
		BeforeAdvice advice = new GreetingBeforeAdvice();
		//Spring提供的代理工厂
		ProxyFactory pf = new ProxyFactory();
		//设置代理目标
		pf.setTarget(target);
		//为代理目标添加增强
		pf.addAdvice(advice);
		//生成代理实例
		Waiter proxy = (Waiter)pf.getProxy();
		proxy.greeTo("John");
		proxy.serveTo("Tom");
	}
}
 
interface Waiter {
	void greeTo(String name);
	void serveTo(String name);
}
 
class NaiveWaiter implements Waiter {
	public void greeTo(String name) {
		System.out.println("greet to " + name + "...");
	}
	public void serveTo(String name) {
		System.out.println("serving " + name + "...");
	}
}
 
class GreetingBeforeAdvice implements MethodBeforeAdvice {
	@Override
	public void before(Method method, Object[] args, Object obj)
			throws Throwable {
		String clientName = (String)args[0];
		System.out.println("How are you! Mr." + clientName + ".");
	}	
}
```

ProxyFactory代理工厂将GreetingBeforeAdvice的增强织入到目标类NaiveWaiter中。在ProxyFactory内部，依然使用的是JDK代理或CGLib代理，将增强应用到目标类中。
Spring定义了org.springframework.aop.framework.AopProxy接口，并提供了两个实现类，Cglib2AopProxy和JdkDynamicAopProxy。



Spring配置的方式（ProxyFactoryBean）


我们通过Spring配置的方式，代替编写ProxyFactory相关的代码。



1.beans.xml内容

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
    xmlns:p="http://www.springframework.org/schema/p"
    xsi:schemaLocation="http://www.springframework.org/schema/beans  http://www.springframework.org/schema/beans/spring-beans-4.0.xsd"
    >
	<bean id="greetingAdvice" class="cl.an.advice.GreetingBeforeAdvice"/>
	<bean id="target" class="cl.an.advice.NaiveWaiter"></bean>
	<bean id="waiter" class="org.springframework.aop.framework.ProxyFactoryBean"
		p:proxyInterfaces="cl.an.advice.Waiter"
		p:interceptorNames="greetingAdvice"
		p:target-ref="target"></bean>
</beans>
```

2.调用代码简化为：

```
public class TestBeforeAdvice {
	public static void main(String[] args) {
		String configPath = "cl/an/advice/beans.xml";
		ApplicationContext ctx = new ClassPathXmlApplicationContext(configPath);
		Waiter waiter = (Waiter)ctx.getBean("waiter");
		waiter.greeTo("John");
		waiter.serveTo("Tom");
	}
}
```

#### 切面（Advisor）方式



本例子中演示的是正则表达式切面，并且限制了只对greeTo方法进行增强。



1.beans.xml的内容

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
    xmlns:p="http://www.springframework.org/schema/p"
    xsi:schemaLocation="http://www.springframework.org/schema/beans  http://www.springframework.org/schema/beans/spring-beans-4.0.xsd"
    >
    <bean id="regexpAdvisor" class="org.springframework.aop.support.RegexpMethodPointcutAdvisor"
    	p:advice-ref="greetingAdvice">
    	<property name="patterns">
    		<list>
    			<!-- 用正则表达式，只对greeTo方法进行增强 -->
    			<value>.*greeT.*</value>
    		</list>
    	</property>	
    </bean>
    <bean id="waiter" class="org.springframework.aop.framework.ProxyFactoryBean"
    	p:interceptorNames="regexpAdvisor"
    	p:target-ref="target"
    	p:proxyTargetClass="true">
    </bean>
	<bean id="greetingAdvice" class="cl.an.advice.GreetingBeforeAdvice"/>
	<bean id="target" class="cl.an.advice.NaiveWaiter"></bean>
</beans>
```

#### 自动创建代理

以上，我们通过ProxyFactoryBean创建织入切面的代理，对于小型系统，可以将就使用，但对拥有众多需要代理Bean的系统系统，需要做的配置工作就太多太多了。

但是，Spring为我们提供了自动代理机制，让容器为我们自动生成代理，把我们从繁琐的配置工作中解放出来。

在内部，Spring使用BeanPostProcessor自动完成这项工作。

我们可以使用BeanNameAutoProxyCreator，只对某些符合通配符的类进行增强，也可以使用DefaultAdvisorAutoProxyCreator，自动扫描容器中的Advisor，并将Advisor自动织入到匹配的目标Bean中，即为匹配的目标Bean自动创建代理。



1.beans.xml的内容

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
    xmlns:p="http://www.springframework.org/schema/p"
    xsi:schemaLocation="http://www.springframework.org/schema/beans  http://www.springframework.org/schema/beans/spring-beans-4.0.xsd"
    >
    <bean id="regexpAdvisor" class="org.springframework.aop.support.RegexpMethodPointcutAdvisor"
    	p:advice-ref="greetingAdvice">
    	<property name="patterns">
    		<list>
    			<!-- 用正则表达式，只对greeTo方法进行增强 -->
    			<value>.*greeT.*</value>
    		</list>
    	</property>	
    </bean>
    <bean id="waiter" class="cl.an.advice.NaiveWaiter"/>
	<bean id="greetingAdvice" class="cl.an.advice.GreetingBeforeAdvice"/>
	<!-- 开启自动切面代理 -->
	<bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"/>
</beans>
```

#### @AspectJ定义切面的方式



这是一种配置最少的切面定义方式，功能强大，但是需要学习AspectJ切点表达式语言。

1.代码内容

```
package cl.an.aspectj;
 
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
 
public class AspectJProxyTest {
	public static void main(String[] args) {
		String configPath = "cl/an/aspectj/beans.xml";
		ApplicationContext ctx = new ClassPathXmlApplicationContext(configPath);
		Waiter waiter = (Waiter)ctx.getBean("waiter");
		waiter.greeTo("John");
		waiter.serveTo("Tom");
	}
}
 
 
interface Waiter {
	void greeTo(String name);
	void serveTo(String name);
}
 
class NaiveWaiter implements Waiter {
	public void greeTo(String name) {
		System.out.println("greet to " + name + "...");
	}
	public void serveTo(String name) {
		System.out.println("serving " + name + "...");
	}
}
 
@Aspect
class PreGreetingAspect {
	@Before("execution(* greeTo(..))")
	public void beforeGreeting() {
		System.out.println("How are you");
	}
}
```

2.beans.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
    xmlns:p="http://www.springframework.org/schema/p"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xsi:schemaLocation="http://www.springframework.org/schema/beans  http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
    http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-4.0.xsd"
    >
	<!-- 开启自动切面代理:基于AspectJ切面的驱动器 -->
	<!-- true:代表使用cglib进行动态代理 -->
	<aop:aspectj-autoproxy proxy-target-class="true" />
 
    <bean id="waiter" class="cl.an.aspectj.NaiveWaiter"/>
	<bean id="greetingAdvice" class="cl.an.aspectj.PreGreetingAspect"/>
</beans>
```

LoadTimeWeaver-LTW


Spring也支持类加载期的织入，在类加载期，通过字节码编辑的技术，将切面织入到目标类中，这种织入方式成为LTW（Load Time Weaving）。就像运行期织入，其底层依靠的是JDK动态代理和CGLib字节码增强一样，类加载器的注入，底层依靠的也是JDK，jang.lang.instrument包中的两个接口（ClasFileTransformer和Instrumentation）。

参考：

http://sexycoding.iteye.com/blog/1062372  

http://blog.csdn.net/scorpio3k/article/details/6745443  

http://blog.csdn.net/qyongkang/article/details/7799603



1.代码

```
package instrument;
 
import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.lang.instrument.Instrumentation;
import java.security.ProtectionDomain;
 
public class Test {
	public static void main(String[] args) {
		System.out.println("I'm in main() of Test...");
	}
}
 
class Transformer implements ClassFileTransformer {
	@Override
	public byte[] transform(ClassLoader loader, String className,
			Class<?> classBeingRedefined, ProtectionDomain protectionDomain,
			byte[] classfileBuffer) throws IllegalClassFormatException {
		System.out.println("Hello " + className + "!");
		return null;
	}	
}
 
//这个代理类，会在程序入口main()方法执行前执行，执行的是premain()方法
class Agent {
	//这个Agent代理类，必须按一下前面方式定义premain()
	public static void premain(String agentArgs,Instrumentation inst) {
		ClassFileTransformer t = new Transformer();
		inst.addTransformer(t);
	}
}
```

2.MANIFEST.MF内容，这个文件在制作jar时用到，会导出到META-INF目录下的MANIFEST.MF

```
Manifest-Version: 1.0
Premain-Class: instrument.Agent
```

注意，这个文件格式要求严格，Premain-Class:后面必须有个空格，文件后面必须至少两行空行。


Eclipse导出jar把，然后执行java -javaagent:test.jar instrument.Test，可以看到Agent的代码内容已经被织入了。



关于Spring LTW，利用的是META-INF目录下的aop.xml，可以利用AspectJ表达式语言进行更多的控制，和前面解释AspectJ的部分差不多。



总结


关于Spring，我个人的理解是：IoC是基础，然后其他一切带给我们编程简便性的地方，全部来源于AOP，让这种横向抽取机制，封装常用操作。比如：

加上@Transactional标记，就对方法或者类开启了事务；

加上@Cacheable标记，就对方法开启了缓存。


# 后记

