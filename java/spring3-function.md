---

layout: single
title: Spring实用功能--Profile、WebService、缓存、消息、ORM
permalink: /java/spring3-function.html

classes: wide

author: Bob Dong

---

# 前言

本篇介绍一些Spring与其他框架结合的实用功能，包括：Apache CXF WebService框架、Redis缓存、RabbitMQ消息、MyBatis框架。

另外对于Profile，也是Spring3.0开始新加的功能，对于开发测试环境、和生产环境分别采用不同的配置，有一定用处。



Profile


Spring3.1新属性管理API：PropertySource、Environment、Profile。

Environment：环境，本身是一个PropertyResolver，但是提供了Profile特性，即可以根据环境得到相应数据（即激活不同的Profile，可以得到不同的属性数据，比如用于多环境场景的配置（正式机、测试机、开发机DataSource配置））。

Profile：剖面，只有激活的剖面的组件/配置才会注册到Spring容器，类似于maven中profile，Spring 3.1增加了一个在不同环境之间简单切换的profile概念, 可以在不修改任何文件的情况下让工程分别在 dev/test/production 等环境下运行。

为了减小部署维护，可以让工程会默认运行在dev模式，而测试环境和生产环境通过增加jvm参数激活 production的profile.

比如，对于如下的一个例子，由于测试环境和生产环境，连接数据库的方式不一样，可以有如下的解决办法：



1、首先ApplicationContext.xml中，xsi:schemaLocation需要引用3.2的xsd

2、ApplicationContext.xml配置如下：

```
<beans profile="production">
	<bean id="dataSourcejdbc" class="org.springframework.jndi.JndiObjectFactoryBean">
		<property name="jndiName" value="java:/MySqlDS_JDBC" />
	</bean>
</beans>
<beans profile="dev">
	<bean id="dataSourcejdbc" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
		<property name="driverClassName" value="com.mysql.jdbc.Driver"/>
		<property name="url" value="jdbc:mysql://IP:3306/db?characterEncoding=utf-8"/>
		<property name="username" value="root"/>
		<property name="password" value="root"/>
	</bean>
</beans>
```

3、开发环境配置，在web.xml中，如下配置：

```
<context-param>  
	<param-name>spring.profiles.default</param-name>  
	<param-value>dev</param-value>  
</context-param>
```

4、生产环境配置

比如，对于Jboss，在bin/run.conf里面，增加启动参数：-Dspring.profiles.active=production

JAVA_OPTS="-Xms2048m -Xmx2048m -XX:MaxPermSize=1024m -Dorg.jboss.resolver.warning=true -Dsun.rmi.dgc.client.gcInterval=3600000 -Dsun.rmi.dgc.server.gcInterval=3600000 -Dsun.lang.ClassLoader.allowArraySyntax=true -Dorg.terracotta.quartz.skipUpdateCheck=true -Dspring.profiles.active=production"



以上是对于Web项目中如何利用profile的一种演示，如果是maven项目，也可以在maven打包时采用不同的profile，命令如下：

mvn clean package -Dmaven.test.skip=true -Ponline

通过P参数采用不同的profile，这样可以实现为开发、测试、生产打出不同的包。



不过，不推荐这种打包方式，应该是对于开发、测试、生产打出一样的包，然后根据机器本身的环境，来决定程序是按照那种环境来运行。

如果公司有根据环境变量的自动化部署方式（比如dev/test/stage/online)，则这个profile是非常管用的。



WebService


Java生态下的WebService框架非常多，apache cxf 是与spring结合最好的一种。配置步骤如下：



1、pom.xml，增加依赖：

```
<dependency>
	<groupId>org.apache.cxf</groupId>
	<artifactId>cxf-rt-frontend-jaxws</artifactId>
	<version>2.7.5</version>
</dependency>
<dependency>
	<groupId>org.apache.cxf</groupId>
	<artifactId>cxf-rt-transports-http</artifactId>
	<version>2.7.5</version>
</dependency>
```

2、web.xml，增加servlet:  

```

<!-- Web Service声明开始 -->
<servlet>
    <servlet-name>cxf</servlet-name>
    <servlet-class>org.apache.cxf.transport.servlet.CXFServlet</servlet-class>
    <load-on-startup>2</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>cxf</servlet-name>
    <url-pattern>/*</url-pattern>
</servlet-mapping>
<!-- Web Service声明结束 -->
```

3、resources目录下，增加applicationContext-cxf.xml，内容如下：

```

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
 xmlns:jaxws="http://cxf.apache.org/jaxws"
 xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
    http://cxf.apache.org/jaxws http://cxf.apache.org/schemas/jaxws.xsd">
 
 <!-- <import resource="classpath:META-INF/cxf/cxf.xml" />
 <import resource="classpath:META-INF/cxf/cxf-servlet.xml" /> -->
 
 <jaxws:endpoint implementor="#basicWebService" address="/BasicWebService" />
</beans>
```

4、BasicWebService来的内容大致如下：

```
@WebService(name = "BasicWebService", serviceName = "BasicWebService", portName = "BasicWebServicePort", targetNamespace = "http://api.domain.com/ws")
@Service
public class BasicWebService {
 @WebMethod
 public void sendHtmlMail(@WebParam(name = "headName") String headName,
   @WebParam(name = "sendHtml") String sendHtml) {
  sendMail.doSendHtmlEmail(headName, sendHtml);
 }
}
```

使用Apache CXF框架，是被Spring容器管理的，也就是说，BasicWebService本身可以设置@Service标记，也可以在BasicWebService中使用@Autowired进行注入。
而其他框架的WebService，比如Jboss直接通过Servlet方式暴露的WebService就不能这样，只能通过一个SpringContextHolder手动从Spring容器中拿，大致如下：

1、首先在web.xml中增加WebService类的servlet，如下：

```
<!-- Web Service声明开始 -->
<servlet>
	<servlet-name>BasicWebService</servlet-name>
	<servlet-class>com.xx.BasisWebService</servlet-class>
</servlet>
<servlet-mapping>
	<servlet-name>BasicWebService</servlet-name>
	<url-pattern>/BasicWebService</url-pattern>
</servlet-mapping>
<!-- Web Service声明结束 --> 
```

2、BasicWebService的内容大致如下：

```

@WebService(name = "BasicWebService", serviceName = "BasicWebService", portName = "BasicWebServicePort", targetNamespace = "http://api.sina.com/ws")
public class BasicWebService {
 
	//这是从Spring容器中拿对象，SpringContextHolder是一个实现了org.springframework.context.ApplicationContextAware的类
	private ISystemConfigService systemConfigService = SpringContextHolder.getBean(ISystemConfigService.class);
 
	@WebMethod
	public String test(@WebParam(name = "inputpara") String inputpara) {
		return inputpara + "_100";
	}
}
```

## Redis



Spring可以简化调用Redis的操作，配置大致如下：



1、pom.xml增加依赖：

```
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-redis</artifactId>
    <version>1.0.6.RELEASE</version>
</dependency>
```

2、resources目录下，增加applicationContext-redis.xml，内容如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:cache="http://www.springframework.org/schema/cache"
       xmlns:p="http://www.springframework.org/schema/p"
       xsi:schemaLocation="http://www.springframework.org/schema/beans  http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
 http://www.springframework.org/schema/cache http://www.springframework.org/schema/cache/spring-cache.xsd">    
    <description>Spring-cache</description>
    <cache:annotation-driven/>
    <bean id="cacheManager" class="org.springframework.data.redis.cache.RedisCacheManager">
        <constructor-arg name="template" index="0" ref="redisTemplate"/>
    </bean>
    <bean id="jedisPoolConfig" class="redis.clients.jedis.JedisPoolConfig">
        <property name="maxActive" value="${redis.pool.maxActive}"/>
        <property name="maxIdle" value="${redis.pool.maxIdle}"/>
        <property name="maxWait" value="${redis.pool.maxWait}"/>
        <property name="testOnBorrow" value="${redis.pool.testOnBorrow}"/>
    </bean>
    <!-- 工厂实现： -->
    <bean id="jedisConnectionFactory" class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory">
        <property name="hostName" value="${redis.ip}"/>
        <property name="port" value="${redis.port}"/>
        <property name="poolConfig" ref="jedisPoolConfig"/>
    </bean>
    <!--模板类： -->
    <bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate" p:connection-factory-ref="jedisConnectionFactory">
        <property name="keySerializer" ref="stringRedisSerializer"/>
    </bean>
    <bean id="stringRedisSerializer" class="org.springframework.data.redis.serializer.StringRedisSerializer"/>
</beans>
```

3、缓存写入参考实现：

```
@Service
public class BrandBaseServiceImpl implements IBrandBaseService {
    @Override
    @Cacheable(value = CacheClientConstant.COMMODITY_BRAND_REDIS_CACHE, key = "'commodity:webservice:all:brand:list'")
    public List<Brand> getAllBrands() {
     try
     {
      List<Brand> brands = brandMapper.getAllBrands();
      return brands;
     } catch (Exception ex)
     {
      logger.error(ex.toString());
      return null;     
     }
    }
    @Override
    @Cacheable(value = CacheClientConstant.COMMODITY_BRAND_REDIS_CACHE, key = "'commodity:webservice:brand:no:'+#brandNo")
    public Brand getBrandByNo(String brandNo) {
        if (StringUtils.isBlank(brandNo))
            return null;
        return brandMapper.getBrandByNo(brandNo);
    }
}
```

4、缓存清除参考实现：

```
@Service
public class RedisCacheUtil {
 
 private final Logger logger = LoggerFactory.getLogger(this.getClass());
    @Autowired
    private RedisTemplate<String,Object> redisTemplate;
    @Autowired
    private JedisConnectionFactory jedisConnectionFactory;    
    @CacheEvict(value = CacheClientConstant.COMMODITY_CATEGORY_REDIS_CACHE, key = "'commodity:webservice:category:no:'+#categoryNo")
    public void cleanCatCacheByNo(String categoryNo)
    {
     List<String> keys = new ArrayList<String>();
        logger.info("[商品服务端]清理分类categoryNo：{}缓存,REDIS SERVER地址：{}", categoryNo, jedisConnectionFactory.getHostName() + ":" + jedisConnectionFactory.getPort());
        if (StringUtils.hasText(categoryNo)) {
         keys.add("commodity:webservice:category:no:" + categoryNo);
            cleanAgain(keys);
        }
    }    
    @CacheEvict(value = CacheClientConstant.COMMODITY_SYSTEMCONFIG_REDIS_CACHE, allEntries = true)
    public void cleanSystemConfigAll()
    {
     logger.info("[商品服务端]清楚SystemConfig缓存");
    }    
    /**
     * 考虑到主从延迟可能会导致缓存更新失效，延迟再清理一次缓存
     * @param keys 需要清除缓存的KEY
     */
    private void cleanAgain(List<String> keys) {
        if (CollectionUtils.isEmpty(keys)) {
            return;
        }
        for (String key : keys) {
            logger.info("清理缓存，KEY：{}", key);
            redisTemplate.delete(key);
        }
    }
}
```

## [RabbitMQ](https://so.csdn.net/so/search?q=RabbitMQ&spm=1001.2101.3001.7020)



Spring也可以简化使用RabbitMQ的操作，配置大致如下：



1、pom.xml增加依赖：

```
<dependency>
	<groupId>org.springframework.amqp</groupId>
	<artifactId>spring-amqp</artifactId>
	<version>${spring.amqp.version}</version>
</dependency>
<dependency>
	<groupId>org.springframework.amqp</groupId>
	<artifactId>spring-rabbit</artifactId>
	<version>${spring.amqp.version}</version>
</dependency>
```

2、发送消息代码例子：

```
@Service
public class MessageSendServiceImpl implements IMessageSendService { 
 private static final String EXCHANGE = "amq.topic"; 
 @Autowired
 private volatile RabbitTemplate rabbitTemplate; 
 private final Logger logger = LoggerFactory.getLogger(this.getClass());
 @Override
 public Boolean sendMessage(String commodityNo) {
  Commodity c = getCommodity(commodityNo);
  // 发送rabbitMQ消息(topic)
  rabbitTemplate.convertAndSend(EXCHANGE, "commodity.update.topic", c);
  logger.info("发送消息成功(topic)：商品编号：" + commodityNo);
  return true;
 } 
}
```

3、resources目录下，增加applicationContext-rabbitmq.xml，用来配置接收消息，内容如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
 xmlns:rabbit="http://www.springframework.org/schema/rabbit" xmlns:task="http://www.springframework.org/schema/task"
 xmlns:context="http://www.springframework.org/schema/context"
 xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
 http://www.springframework.org/schema/rabbit http://www.springframework.org/schema/rabbit/spring-rabbit-1.1.xsd
 http://www.springframework.org/schema/task http://www.springframework.org/schema/task/spring-task.xsd
 http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">
 <!-- 定义rabbitmq连接工厂，生产环境使用集群配置，支持failover，rabbitmq.host=192.168.211.230:5672 -->
 <rabbit:connection-factory id="connectionFactory" addresses="${rabbitmq.host}" />
 <rabbit:admin connection-factory="connectionFactory" />
 <rabbit:template id="amqpTemplate" connection-factory="connectionFactory" channel-transacted="true"
  message-converter="jsonMessageConverter" />
 <bean id="jsonMessageConverter" class="org.springframework.amqp.support.converter.JsonMessageConverter">
  <property name="classMapper">
   <bean class="org.springframework.amqp.support.converter.DefaultClassMapper">
   </bean>
  </property>
 </bean>
 <!--
    两种业务需求：
    1. 同一个服务部署在多台服务器上，如果想消息被一个服务收取，则要配置name，<rabbit:listener 里的queues=这里的name
    2. 同一个服务部署在多台服务器上，如果想消息被所有的服务收取，刚不要配置name，用rabbitmq自动创建的匿名name，这时要去掉这里的name属性， 并且<rabbit:listener里的queues=这里的id
    一般来说，都是第一种业务需求较多
 -->    
    <rabbit:queue id="queue的id，可以和name一样" name="queue的名字，在rabbitmq控制台可以看到，例如commodity.update.topic.queue">
        <rabbit:queue-arguments>
            <entry key="x-ha-policy" value="all" />
        </rabbit:queue-arguments>
    </rabbit:queue> 
 <!-- CONSUMER -->
    <!-- 这里的error-handler最好都配置，因为rabbitmq报的异常默认是不被捕获的，如果这里没有error-handler,log级别又没指定到amqp的包，那么错误将不会被察觉    -->
 <rabbit:listener-container connection-factory="connectionFactory" message-converter="jsonMessageConverter" 
            channel-transacted="true" error-handler="rabbitMqErrorHandler" concurrency="10"
               auto-startup="true">
        <rabbit:listener queues="rabbit:queue中定义的name或者id" ref="commodityUpdateListener" method="handleMessage" />
    </rabbit:listener-container>
    <rabbit:topic-exchange name="amq.topic" >
        <rabbit:bindings>
            <!-- 这里的queue是<rabbit:queue 里的ID -->            
            <rabbit:binding pattern="发送方的routingKey，对于上面的发送就是commodity.update.topic" queue="queue的名字，在rabbitmq控制台可以看到，例如commodity.update.topic.queue"/>
        </rabbit:bindings>
    </rabbit:topic-exchange>
</beans>
```

4、接收消息代码例子：

```
@Component
public class CommodityUpdateListener {
 public void handleMessage(Commodity commodity) {
  if(commodity==null)
  {
   logger.info("XXX");
   return;
  }
  //处理逻辑
 }
}
```

5、处理消息错误代码例子：

```

@Component
public class RabbitMqErrorHandler implements ErrorHandler {
 
 private static Logger logger = LoggerFactory.getLogger(RabbitMqErrorHandler.class);
 
 @Override
 public void handleError(Throwable t) {
  logger.error("Receive rabbitmq message error:{}", t); 
 }
}
```

MyBatis


Spring可以大大简化使用MyBatis这种ORM框架，定义出接口和Mapper文件之后，Spring可以自动帮我们生成实现类。我曾经在DotNet框架下使用过MyBatis.Net，所有的Mapper的实现类都需要手工写代码，而Spring帮我节省了很多编码工作量。

大致配置步骤如下：

1、pom.xml增加依赖：

```

<dependency>
	<groupId>org.mybatis</groupId>
	<artifactId>mybatis-spring</artifactId>
	<version>1.1.1</version>
</dependency>
<dependency>
	<groupId>org.mybatis.caches</groupId>
	<artifactId>mybatis-ehcache</artifactId>
	<version>1.0.1</version>
</dependency>
```

2、resources目录下，applicationContext.xml中，一般放置关于mybatis的配置，内容如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
 xmlns:jee="http://www.springframework.org/schema/jee" xmlns:tx="http://www.springframework.org/schema/tx"
 xmlns:context="http://www.springframework.org/schema/context" xmlns:task="http://www.springframework.org/schema/task"
 xmlns:aop="http://www.springframework.org/schema/aop"
 xsi:schemaLocation="http://www.springframework.org/schema/beans  http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
 http://www.springframework.org/schema/tx  http://www.springframework.org/schema/tx/spring-tx-3.2.xsd
 http://www.springframework.org/schema/jee  http://www.springframework.org/schema/jee/spring-jee-3.2.xsd
 http://www.springframework.org/schema/context  http://www.springframework.org/schema/context/spring-context-3.2.xsd
 http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-3.2.xsd">
 <description>Spring公共配置</description>
 <!--开启注解 -->
 <context:annotation-config />
 <!-- 开启自动切面代理 -->
 <aop:aspectj-autoproxy /> 
 <context:component-scan base-package="com.xx">
  <context:exclude-filter type="annotation" expression="org.springframework.stereotype.Controller" />
 </context:component-scan>
 <!-- 定义受环境影响易变的变量 -->
 <bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
  <property name="systemPropertiesModeName" value="SYSTEM_PROPERTIES_MODE_OVERRIDE" />
  <property name="ignoreResourceNotFound" value="true" />
  <property name="locations">
   <list>
    <!-- 标准配置 -->
    <value>classpath*:/application.properties</value>
    <value>classpath*:/config.properties</value>
    <!-- 本地开发环境配置 -->
    <value>file:/d:/conf/pcconf/*.properties</value>
    <!-- 服务器生产环境配置 -->
    <value>file:/etc/conf/pcconf/*.properties</value>
   </list>
  </property>
  <!--property name="ignoreUnresolvablePlaceholders" value="true" / -->
 </bean>
 <tx:annotation-driven transaction-manager="transactionManager" proxy-target-class="true" />
 <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
     <property name="dataSource" ref="dataSourcejdbc"/>    
    </bean>
 <!-- 强烈建议用JdbcTemplate代替JdbcUtils -->
 <bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
  <property name="dataSource" ref="dataSourcejdbc" />
 </bean>
 <bean id="sqlSessionFactoryWrite" class="org.mybatis.spring.SqlSessionFactoryBean">
  <property name="dataSource" ref="dataSourcejdbc" />
 </bean>
 <!-- 会自动将basePackage中配置的包路径下的所有带有@Mapper标注的Dao层的接口生成代理，替代原来我们的Dao实现。 -->
 <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
  <property name="sqlSessionFactory" ref="sqlSessionFactoryWrite" />
  <property name="basePackage" value="com/xx/pc/template" />
 </bean> 
 <beans profile="production">
  <bean id="dataSourcejdbc" class="org.springframework.jndi.JndiObjectFactoryBean">
   <property name="jndiName" value="java:/MySqlDS_JDBC" />
  </bean>
 </beans>
    <beans profile="dev">
     <bean id="dataSourcejdbc" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
         <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
      <property name="url" value="jdbc:mysql://ip:3306/dbname?characterEncoding=utf-8"/>
      <property name="username" value="root"/>
      <property name="password" value="root"/>
     </bean>
    </beans> 
</beans>
```

3、定义接口，及在src/main/resource对应接口的包路径下定义同名的xml配置文件即可。



Spring初始化完毕后，会自动帮我们生成Mapper的实现类。

# 后记

