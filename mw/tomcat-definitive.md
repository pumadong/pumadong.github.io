---

layout: single
title: 读一本书《Tomcat权威指南》
permalink: /mw/tomcat-definitive.html

classes: wide

author: Bob Dong

---

# 前言

本篇是《Tomcat权威指南》第二版学习笔记，Jason Brittain著，英文名是：Tomcat:The Definitive Guide，中国电力出版社，2009.9出版。

在工作中经常使用Tomcat、JBoss、Jetty等Java容器，但都不曾系统的学习总结过，本次拿出一个周末的时间，通过本书，较为系统的学习一下Tomcat，并结合互联网的参考资料，写下这篇学习总结，感觉还是受益良多。

关于《Tomcat权威指南》第二版这本书，是基于Tomcat 6的，翻译不是很专业，一些计算机技术术语翻译的不够准确，印刷中也有一些错别字，所以对于没有基础的初学者，容易被误导，如果有一定基础，把这本书系统看一遍，作为工具书，还是不错的。

关于本书的原作者，是spigit.com的软件架构师，对Tomcat作为Web服务器的性能估计较为乐观，这个乐观的估计没有得到大数据量高并发系统的验证；相反，仅仅把Tomcat作为Java容器，甚至仅仅作为开发过程Java容器，生产过程使用JBoss的案例貌似更多。

关于选择这本书的原因，是市场上系统的讲解Tomcat的书实在是少，相对来说，从系统化讲解来说，这本书还不错

# Tomcat目录结构

| 目录    | 说明                                                         |
| ------- | ------------------------------------------------------------ |
| bin     | 存放启动、停止服务器的脚本文件                               |
| conf    | 存放服务器的配置文件、最重要的是server.xml文件               |
| lib     | 存放jar文件，服务器和所有的web应用程序都可以访问             |
| logs    | 存放服务器的日志文件                                         |
| temp    | 存放Tomcat运行时的临时文件                                   |
| webapps | 缺省的web应用的发布目录，在server.xml中的“Host appBase="webapps"...”节点定义 |
| work    | Tomcat的工作目录，默认情况下把编译JSP文件生成的servlet类文件放于此目录下 |

# Tomcat conf目录下的配置文件

| 文件                | 说明                                                         |
| ------------------- | ------------------------------------------------------------ |
| server.xml          | Tomcat主配置文件                                             |
| web.xml             | servlet与其他适用于整个Web应用程序的配置，必须符合servlet规范 |
| tomcat-users.xml    | Tomcat的UserDatabaseRealm用于认证的默认角色、用户及密码清单  |
| catalina.policy     | Tomcat的Java安全防护策略文件                                 |
| context.xml         | 默认的context设置，应用于安装了Tomcat的所有主机的所有部署内容 |
| catalina.properties |                                                              |
| logging.properties  |                                                              |

# Tomcat lib目录下的jar包作用

| jar                      | 作用                                                         |
| ------------------------ | ------------------------------------------------------------ |
| annotation-api.jar       | JavaEE annotations classes                                   |
| catalina.jar             | Implementation of the Catalina servlet container portion of Tomcat |
| catalina-ant.jar         | Tomcat Catalina Ant tasks                                    |
| catalina-ha.jar          | 高可用package                                                |
| catalina-storeconfig.jar |                                                              |
| catalina-tribes.jar      | 组通信package                                                |
| ecj-4.3.1.jar            | Eclipse JDT Java compiler，即Eclipse开发的Jsp编译器，Tomcat5.5开始，开始默认使用这个编译器编译Jsp页 |
| el-api.jar               | expression language                                          |
| jasper.jar               | Tomcat Jasper JSP Compiler and Runtime，Jasper 2 JSP Engine to implement the JavaServer Pages 2.1 specification |
| jasper-el.jar            | Tomcat Jasper EL implementation                              |
| jsp-api.jar              | 在Tomcat6/7/8中，版本分别是2.1/2.2/2.3                       |
| servlet-api.jar          | 在Tomcat6/7/8中，版本分别是2.5/3.0/3.1                       |
| tomcat-api.jar           | Several interfaces defined by Tomcat                         |
| omcat-coyote.jar         | Tomcat connectors and utility classes                        |
| tomcat-dbcp.jar          | Database connection pool implementation，based on package-renamed copy of Apache Commons Pool and Apache Commons DBCP |
| tomcat-i18n-es.jar       | Optional JARs containing resource bundles for other languages。As default bundles are also included in each individual JAR，they can be safely removed if no internationalization of messages is needed |
| tomcat-i18n-fr.jar       |                                                              |
| tomcat-i18n-fa.jar       |                                                              |
| tomcat-jdbc.jar          | An alternative database connection pool implementation，known as Tomcat JDBC pool。See documentation for more details |
| tomcat-jni.jar           |                                                              |
| tomcat-spdy.jar          |                                                              |
| tomcat-util.jar          | Common classes used by various components of Apache Tomcat   |
| tomcat-util-scan.jar     |                                                              |
| tomcat-websocket.jar     | 在Tomcat7中这个包叫tomcat7-websocket.jar                     |
| websocket-api.jar        |                                                              |

# Tomcat部署的两种方法

（1）server.xml Context部署：在server.xml文件中增加一个Context元素

（2）Context XML片段文件部署：在Tomcat的CATALINA_HOME/conf/[EngineName]/[Hostname]目录中增加一个新的Context XML片段文件



Tomcat性能优化


性能指标：吞吐量


Responsetime、CpuLoad、MemoryUsage



压力测试工具


a、Apache Benchmark（ab，内含在Apache httpd Web服务器的发行版中，网址为：http://httpd.apache.org)    ab -k -n 100000 -c 149 http://ip:8080

b、Siege（http://www.joedog.org/JoeDog/Siege） siege -b -r 671 -c 149 tomcathost:8080

c、Apache Jakarta的JMeter（http://jakarta.apache.org/jmeter），据说易用、灵活、图形化、报表性强、但是单纯测试“每秒钟请求并完成非常大量HTTP请求”方面，不如以上两种



Tomcat连接器


Tomcat提供了3种不同的服务器设计实现方法，JIO（java.io）：这是Tomcat默认的连接器实现办法，也成为“Coyote”，是使用java.io核心网络类的纯Java TCP实现；APR（Apache Portable Runtime），使用了libtcnative库（c语言编写的），适用于HTTPS链接；NIO（java.nio），非阻塞的方式，更少的线程



Apache连接器


mod_jk、mod_proxy_ajp、mod_proxy_http



JVM优化


所有的Java相关应用的JVM调优，其原理都是一致的，请参考这里：http://blog.csdn.net/puma_dong/article/details/12529905#t4



Tomcat优化


a、禁用DNS查询，在Connection节点中，增加enableLookups="false"，这个设置会导致getRemoteHost()只能获取到IP地址；

b、调整线程数，maxThreads默认是200，示例：

<Connector port="80" protocol="HTTP/1.1" maxThreads="1000" minSpareThreads="200" maxSpareThreads="800" acceptCount="800" connectionTimeout="20000" enableLookups="false" redirectPort="8443"/>



maxThrads和acceptCount，这两个值如何起作用，请看下面三种场景：

a、接受一个请求，此时tomcat启动的线程数没有到达maxThreads，tomcat会启动一个线程来处理此请求；

b、接受一个请求，此时tomcat启动的线程数已经到达maxThreads，tomcat会把此请求放入等待队列，等待空闲线程；

c、接受一个请求，此时tomcat启动的线程数已经到达maxThreads，等待队列中得请求个数也达到了acceptCount，此时tomcat会直接拒绝此次请求，返回connection refused



maxThreads如何配置：

一般的服务器操作都包含量方面：1计算（主要消耗cpu），2等待（io、数据库等）；

第一种极端情况，如果我们的操作是纯粹的计算，那么系统响应时间的主要限制就是cpu的运算能力，此时maxThreads应该尽量设的小，降低同一时间内争抢cpu的线程个数，可以提高计算效率，提高系统的整体处理能力；

第二种极端情况，如果我们的操作纯粹是IO或者数据库，那么响应时间的主要限制就变为等待外部资源，此时maxThreads应该尽量设的大，这样才能提高同时处理请求的个数，从而提高系统整体的处理能力。此情况因为tomcat同时处理的请求量会比较大，所以需要关注一下tomcat的虚拟机内存设置和linux的open file限制；

如果maxThreads设置过大，比如5000，由于cpu就需要在多个线程之间来回切换，以保证每个线程都会获得cpu时间，即通常我们说的并发执行，反而降低了cpu效率，直接导致响应时间急剧增加，所以maxThreads的配置绝对不是越大越好；

现实应用中，我们的操作都会包含以上两种类型（计算、等待），所以maxThreads的配置并没有一个最优值，一定要根据具体情况来配置；

最好的做法是：在不断测试的基础上，不断调整、优化，才能得到最合理的配置。



acceptCount如何配置：

可以设置跟maxThreads一样大，这个值应该是主要根据应用的访问峰值与平均值来权衡配置的；

如果设的较小，可以保证接受的请求较快响应，但是超出的请求可能就直接被拒绝；

如果设的较大，可能就出现大量的请求超时的情况，因为我们系统的处理能力是一定的。



Tomcat和Apache的区别


（1）apache：是Web服务器，本身只支持静态网页，但可以通过插件扩展支持PHP、Java等

（2）tomcat：是应用服务器，它只是一个servlet容器，是Apache的扩展，较少直接用作Web服务器



Tomcat鸡肋功能


（1）Tomcat支持多实例部署，即在一个Tomcat安装的基础上，通过配置多个实例，达到bin/lib等目录共享，及版本统一的目的，实际如果想多实例，直接在生产中拷贝两套Tomcat反而更简单快捷

（2）Tomcat Admin管理Web界面，实际的生产中，不会用这个功能，并且应该是把这些功能从生产部署中删除的，达到安全的目的

（3）热部署，实际的生产中，更多的是冷却部署



总结


原书作者观点


原书作者的观点是Tomcat处理静态页面和图片的性能也是优于Apache的，他是通过一个不完全的测试案例：150个线程内（Tomcat默认运行的线程数），对小文件和小图片进行压力测试得出这个结论，所以他推崇“既把Tomcat作为Java容器，也直接用作Web服务器”，在中小应用中，这可能没有问题，但是在大型/超大型互联网应用中，Apache/Nginx的性能是被广泛证明了的，Apache/Nginx+Tomcat的组合依然是更优的选择。



关于本书


关于Ant的内容，没有去看，我用的是Maven；

关于Apache的内容，没有去看，我用的是Nginx；

关于权限相关，没有去看，应用场景很少，权限还是整体交给运维的好；

关于集群，没有去看，个人不觉得Tomcat本身的集群有什么用。



学习总结


在工作中经常使用Tomcat、JBoss、Jetty等Java容器，但都不曾系统的学习总结过，本次拿出一个周末两天的时间，通过本书，较为系统的学习一下Tomcat，并结合互联网的参考资料，写下这篇学习总结，感觉还是受益良多。

关于《Tomcat权威指南》第二版这本书，是基于Tomcat6的，翻译不是很专业，一些计算机技术术语翻译的不够准确，印刷中也有一些错别字，所以对于没有基础的初学者，容易被误导，如果有一定基础，把这本书系统看一遍，作为工具书，还是不错的。

关于本书的原作者，是spigit.com的软件架构师，对于Tomcat作为Web服务器的性能评估较为乐观，这个乐观的估计没有得到大数据量高并发系统的验证；相反，仅仅把Tomcat作为Java容器，似乎更为妥当。



Tomcat日志


Tomcat日志，默认情况下，在启动的时候，会产生当天的文件，比如，catalina.2014-08-14.log，但是启动完毕后，之后的控制台日志就只往catalina.out文件里面写了，为了每天产生一个日志，有多种方式：

1、官方网站：http://tomcat.apache.org/tomcat-7.0-doc/logging.html

2、网上多有介绍使用cronolog进行切割的方式

3、下面介绍配置Solr时，使用的Log4j的配置方式

把jcl-over-slf4j-1.6.6.jar、jul-to-slf4j-1.6.6.jar、log4j-1.2.16.jar、slf4j-api-1.6.6.jar、slf4j-log4j12-1.6.6.jar、log4j.properties 6个文件放在Tomcat安装目录/lib下面即可产生每天一个的cata.yyyy-mm-dd.log日志文件，log4j.properties内容如下（这个文件实际是在：\solr-4.9.0\example\resources 目录下面的）：

```
#  Logging level
log4j.rootLogger=INFO, file, CONSOLE

log4j.appender.CONSOLE=org.apache.log4j.ConsoleAppender

log4j.appender.CONSOLE.layout=org.apache.log4j.PatternLayout
log4j.appender.CONSOLE.layout.ConversionPattern=%-4r [%t] %-5p %c %x \u2013 %m%n

#- size rotation with log cleanup.
log4j.appender.file=org.apache.log4j.RollingFileAppender
log4j.appender.file.MaxFileSize=4MB
log4j.appender.file.MaxBackupIndex=9

#- File to log to and log format
log4j.appender.file.File=logs/solr.log
log4j.appender.file.layout=org.apache.log4j.PatternLayout
log4j.appender.file.layout.ConversionPattern=%-5p - %d{yyyy-MM-dd HH:mm:ss.SSS}; %C; %m\n
```



# 后记