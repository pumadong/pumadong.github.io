---

layout: single
title: 读一本书《Spring3.X企业应用开发实战》--DAO和事务
permalink: /java/spring3-dao.html

classes: wide

author: Bob Dong

---

# 前言

本篇是《Spring3.X企业应用开发实战》，陈雄华 林开雄著，电子工业出版社，2012.2出版”的学习笔记的第二篇，关于DAO和事务。

本篇从DAO操作，以及事务处理的基本知识谈起，介绍事务本身，以及Spring如何通过注解实现事务。



DAO


近几年持久化技术领域异常喧嚣，各种框架如雨后春笋般地冒出，Sun也连接不断的颁布了几个持久化规范。

Spring对多个持久化技术提供了持久化支持，包括Hibernate,iBatis,JDO,JPA,TopLink，另外，还通过Spring JDBC框架对JDBC API进行简化。

Spring面向DAO制定了一个通用的异常体系，屏蔽具体持久化技术的异常，使业务层和具体的持久化技术达到解耦。

此外，Spring提供了模板类简化各种持久化技术的使用。

通用的异常体系及模板类是Spring整合各种五花八门持久化技术的不二法门，Spring不但借此实现了对多种持久化技术的整合，还可以不费吹灰之力整合潜在的各种持久化框架，体现了“开-闭原则”的经典应用。

Spring支持目前大多数常用的数据持久化技术，Spring定义了一套面向DAO层的异常体系，并为各种支持的持久化技术提供了异常转换器。这里，我们在设计DAO接口时，就可以抛开具体的实现技术，定义统一的接口。

不管采用何种持久化技术，访问数据的流程是相对固定的。Spring将数据访问流程划分为固定和变化两部分，并以模板的方式定义好流程，用回调接口将变化的部分开放出来，留给开发者自行定义。这样，我们仅需提供业务相关的逻辑就可以完成整体的数据访问了。



4种配置数据源的办法


1、DHCP数据源

org.apache.commons.dbcp.BasicDataSource，需要commons-dhcp.jar和commons-pool.jar


对于MySQL，如果数据源配置不当，将可能发生经典的“8小时问题”:

原因是MySQL在默认情况下，如果发现一个连接的空闲时间超过8小时，将会在数据库端自动关闭这个连接。而数据源并不知道这个连接以及被数据库关闭了，当它将这个无用的连接返回给某个DAO时，DAO就会报无法获取Connection的异常。

如果采用DHCP的默认配置，由于testOnBorrow属性的默认值为true，数据源将在连接交给DAO前，检测这个链接是否是好的，如果连接有问题（在数据库端被关闭），则会取一个其他的连接给DAO。所以，并不会有“8小时问题”。如果每次将连接交给DAO时都检测连接有效性，在高并发的应用中将带来性能的问题，因为它会需要更多的数据库访问请求。

一种推荐的高效方式是：将testOnBorrow设置为false，而将testWhileIdle设置为true，再设置好timeBetweenEvictionRunsMillis值。这样，DBCP将通过一个后台线程定时对空闲连接进行检测，当发现无用的连接时，就会将他们清除掉。只要这个值小于8小时，就可以避免“8小时问题”。

当然， MySQL本身可以通过调整interactive-timeout（以秒为单位）配置参数，更改空闲连接的过期时间。所以，设置这个timeBetweenEvictionRunsMillis值时，必须首先获知MySQL空闲连接的最大过期时间。

8小时问题的其他参考：http://blog.csdn.net/wangfayinn/article/details/24623575



2.C3P0数据源，它在lib目录中与Hibernate一起发布。

很多生产项目使用这个数据源的连接池，推荐。



3、Spring的数据源实现类

org.springframework.jdbc.datasource.DriverManagerDataSource

这个数据源是没有连接池的，比较适合单元测试或简单的独立应用使用



4、使用应用服务器的JNDI数据源

org.springframework.jndi.JndiObjectFactoryBean

一般是在开发过程中使用Spring的数据源实现类就可以了，有公司也在测试生产环境中使用JNDI数据源。

不过这种方式，会对容器做一些改动，不是一种最好的方式，个人推荐C3P0。 



一个JBOSS的JNDI数据源文件例子

一个JBOSS的JNDI数据源文件mysql-ds.xml，需要拷贝至{jboss_home}/server/{default}/deploy/，内容如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<!-- See http://www.jboss.org/community/wiki/Multiple1PC for information about local-tx-datasource -->
<!-- $Id: mysql-ds.xml 88948 2009-05-15 14:09:08Z jesper.pedersen $ -->
<!--  Datasource config for MySQL using 3.0.9 available from:
http://www.mysql.com/downloads/api-jdbc-stable.html
-->
<datasources>
  <local-tx-datasource>
    <jndi-name>MySqlDSSlave1</jndi-name>
    <connection-url>jdbc:mysql://ip:3306/dbname?useUnicode=true&characterEncoding=utf-8</connection-url>
    <driver-class>com.mysql.jdbc.Driver</driver-class>
    <user-name>root</user-name>
    <password>root</password>
    <exception-sorter-class-name>org.jboss.resource.adapter.jdbc.vendor.MySQLExceptionSorter</exception-sorter-class-name>
    <min-pool-size>5</min-pool-size>
    <max-pool-size>150</max-pool-size>
    <!-- should only be used on drivers after 3.22.1 with "ping" support-->
    <valid-connection-checker-class-name>org.jboss.resource.adapter.jdbc.vendor.MySQLValidConnectionChecker</valid-connection-checker-class-name>
    <!-- sql to call when connection is created
    <new-connection-sql>some arbitrary sql</new-connection-sql>
    -->
    <!-- sql to call on an existing pooled connection when it is obtained from pool - MySQLValidConnectionChecker is preferred for newer drivers-->
    <check-valid-connection-sql>select 1</check-valid-connection-sql>
    <!-- corresponding type-mapping in the standardjbosscmp-jdbc.xml (optional) -->
    <metadata>
       <type-mapping>mySQL</type-mapping>
    </metadata>
  </local-tx-datasource>
  <no-tx-datasource>
    <jndi-name>MySqlDS_JDBC</jndi-name>
    <connection-url>jdbc:mysql://ip:3306/dbname?useUnicode=true&characterEncoding=utf-8</connection-url>
    <driver-class>com.mysql.jdbc.Driver</driver-class>
    <user-name>root</user-name>
    <password>root</password>
    <exception-sorter-class-name>org.jboss.resource.adapter.jdbc.vendor.MySQLExceptionSorter</exception-sorter-class-name>
    <min-pool-size>5</min-pool-size>
    <max-pool-size>150</max-pool-size>
    <!-- should only be used on drivers after 3.22.1 with "ping" support-->
    <valid-connection-checker-class-name>org.jboss.resource.adapter.jdbc.vendor.MySQLValidConnectionChecker</valid-connection-checker-class-name>
    <!-- sql to call when connection is created
    <new-connection-sql>some arbitrary sql</new-connection-sql>
    -->
    <!-- sql to call on an existing pooled connection when it is obtained from pool - MySQLValidConnectionChecker is preferred for newer drivers-->
    <check-valid-connection-sql>select 1</check-valid-connection-sql>
    <!-- corresponding type-mapping in the standardjbosscmp-jdbc.xml (optional) -->
    <metadata>
       <type-mapping>mySQL</type-mapping>
    </metadata>
  </no-tx-datasource>
</datasources>
```

Spring Data


Spring有一个比较活跃的子项目，名为SpringData，这个项目的目标主要是让访问No-SQL更加方便，并支持map-reduce框架和云计算的数据服务。

第二个目标就是支持基于关系型数据库的数据服务，如Oracle RAC。对于拥有海量数据的项目，可以用SpringData这样的项目来简化项目的开发，SpringData会让数据的访问变得更加方便。SpringData由多个子项目组成，支持CouchDB、MongoDB、Neo4J、Hadoop、Hbase、Cassandra等，有兴趣的读者可以关注：http://www.springsource.com/spring-data。



事务


何为数据库事务


数据库事务有严格的定义，它必须同时满足4个特性：原子性（Atomic）、一致性（Consistency）、隔离性（Isolation）和持久性（Durabiliy），简称为ACID。

原子性：表示组成一个事务的多个数据库操作是一个不可分割的原子单元，只有所有的操作执行成功，整个事务才提交，事务中任何一个数据库操作失败，已经执行的任何操作都必须撤销，让数据库返回到初始状态。

一致性：事务操作成功后，数据库所处的状态和它的业务规则是一致的，即数据不会被破坏。如从A账户转账100元到B账户，不管操作成功与否，A和B的存款总额是不变的。

隔离性：在并发数据操作时，不同的事务拥有各自的数据空间，他们的操作不会对对方产生干扰。准确地说，并非要求做到完全无干扰，数据库规定了各种事务隔离级别，不同隔离级别对应不同的干扰程度，隔离级别越高，数据一致性越好，但并发性越弱。

持久性：一旦数据提交成功后，事务中所有的数据操作都必须被持久化到数据库中，即时提交事务后，数据库马上崩溃，在数据库重启时，也必须能保证能够通过某种机制恢复数据。

数据库管理系统一般采用重执行日志保证原子性、一致性和持久性，重执行日志记录了数据库变化的每一个动作，数据库在一个事务中执行一部分操作后发生错误退出，数据即可以根据重执行日志撤销已经执行的操作。此外，对于已经提交的事务，即使数据库崩溃，在重启数据库时也能够根据日志对尚未持久化的数据进行相应的重执行操作。

和Java程序采用对象锁机制进行线程同步类似，数据库管理系统采用数据库锁机制保证事务的隔离性。当多个事务试图对相同的数据进行操作时，只有持有锁的事务才能操作数据，直到前一个事务完成后，后面的事务才有机会对数据执行操作。



数据并发问题


一个数据库拥有多个访问客户端，这些客户端都可以并发方式访问数据库。

数据库中相同数据可能同时被多个事务访问，如果没有采取必要的隔离措施，就会导致各种并发问题，破坏数据的完整性。

这些问题可以归结为5类，包括3类数据读问题（脏读、不可重复读、幻象读）以及2类数据更新问题（第一类丢失更新和第二类丢失更新）。

| 类型                            | 描述                                                         | 场景                                                         | 解决                                |
| ------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ----------------------------------- |
| 脏读（dirty read）              | A事务读取B事务商务未提交的更改数据，并在这个数据的基础上操作 | 事务1：张三给李四汇款100元，记录日志（假设10秒）<br/>事务2：此时李四查询，发现到账<br/>事务1：记录日志出错，回滚，钱退给张三<br/><br/>结果：导致李四错误的认为收到款项 | 把事务隔离级别调整到READ COMMITTED  |
| 不可重复读（unrepeatable read） | 是指A事务读取了B事务已经提交的更改数据，导致A事务对于同一个数据的多次读取，结果是不一样的。<br/><br/>一个事务先后读取同一条记录，但两次读取的数据不同，我们称之为不可重复读。 | 事务1：张三先后两次查询某一账号的余额<br/>事务2：在两次查询期间，李四完成了银行转账<br/><br/>结果：导致两次的查询结果不同 | 把事务隔离级别调整到REPEATABLE READ |
| 幻象读（phantom read）          | A事务读取B事务提交的新增数据，一般发生在计算统计数据的事务中。<br/><br/>一个事务先后读取一个范围的记录，但两次读取的纪录数不同，我们称之为幻象读。 | 事务1：张三先后两次查询所有账号的余额<br/>事务2：在两次查询期间，李四给张三新开一个账号，并存入100元<br/><br/>结果：导致两次查询的结果不同 | 把事务隔离级别调整到SERIALIZABLE    |

参考：http://www.cnblogs.com/Sun_Blue_Sky/articles/2139996.html



不可重复读和幻象读，前者是指读到了已经提交事务的更改数据，后者是指读到了其他已经提交的事务的新增数据，为了避免这两种情况，采取的对策是不同的，对于前者，需要锁行，对于后者，需要锁表。

第一类丢失更新：A事务撤销时，把已经提交的B事务的更新覆盖了。

第二类丢失跟新：A事务提交时，把已经提交的B事务的更新覆盖了。



数据隔离级别

| 隔离级别        | 脏读   | 不可重复读 | 幻象读 | 第一类丢失更新 | 第二类丢失更新 |
| --------------- | ------ | ---------- | ------ | -------------- | -------------- |
| READ UNCOMMITED | 允许   | 允许       | 允许   | 不允许         | 允许           |
| READ COMMITED   | 不允许 | 允许       | 允许   | 不允许         | 允许           |
| REPEATABLE READ | 不允许 | 不允许     | 允许   | 不允许         | 不允许         |
| SERIALIZABLE    | 不允许 | 不允许     | 不允许 | 不允许         | 不允许         |

SqlServer2008R2的默认隔离级别是“READ COMMITED”，MySQL的默认隔离级别是“REPEATABLE READ”。



Spring 事务


Spring声明式事务是Spring最核心、最常用的功能。

由于Spring通过IOC和AOP的功能非常透明地实现了声明式事务的功能，一般的开发者基本上无须了解Spring声明式事务的内部细节，仅需要懂得如何配置就可以了。

但是在实际应用开发过程中，Spring的这种透明地高阶封装在带来便利的同时，也给我们带来了困惑。

就像通过流言传播的消息，最终听众已经不清楚事情的真相了，而这对于应用开发开说是很危险的。

剖析实际应用中给开发者造成困惑的各种观点，通过分析Spring事务管理的内部运作机制将真相还原出来，真相如下：

在没有事务管理的情况下，DAO照样可以顺利进行数据操作；

将应用分出Web、Service及DAO层只是一种参考的开发模式，并非是事务管理工作的前提条件；

Spring通过事务传播机制可以很好地应对事务方法嵌套调用的情况，开发者无需为了事务管理而可以改变服务方法的设计；

由于单实例的对象不存在线程安全问题，所以经过事务管理增强的单实例Bean可以很好地工作在多线程环境下；

混合使用多个数据访问技术框架的最佳组合是一个ORM技术框架（如Hibernate或JPA等）+一个JDBC技术框架（如Spring JDBC或iBatis)。直接使用ORM技术框架对应的事务管理器就可以了，但必须考虑ORM缓存同步的问题。

Spring AOP增强有两个方案：其一为基于接口的动态代理，其二为基于CGLib动态生成子类的代理。由于Java语法的特性，有些特殊方法不能被Spring AOP代理，所以也就无法享受AOP织入带来的事务增强；

使用Spring JDBC时如果直接获取Connection，可能会造成连接泄露。为降低连接泄露的可能性，尽量使用DataSourceUtils获取数据连接。也可以对数据源进行代理，以便使数据源具有感知事务上下文的能力。

我们描述了Spring JDBC防止连接泄露的解决方案，Spring同样把这种解决方案平滑应用到其他的数据访问技术框架中。



DataSourceUtils


Spring提供了一个能从当前事务上下文中获取绑定的数据连接的工具类，那就是DataSourceUtils。

Spring强调必须使用DataSourceUtils工具类获取数据连接，Spring的JdbcTemplate内部也是通过DataSourceUtils来获取连接的。

是否使用DataSourceUtils获取数据连接就可以高枕无忧了呢？理想很美好，但现实很残酷：如果DataSourceUtils在没有事务上下文的方法中使用getConnection()获取连接，依然会造成数据连接泄露！

而通过分析JdbcTemplate模板的实现，发现JdbcTemplate严谨的获取连接及释放连接的模式化流程保证了JdbcTemplate对数据连接泄露问题的免疫性。

所以，如果有可能，请尽量使用JdbcTemplate、HibernateTemplate等这些模板进行数据访问操作，避免直接获取数据连接的操作。

如果不得已要显式获取数据连接，除了使用DataSourceUtils获取事务上下文绑定的连接之外，还可以通过TransactionAwareDataSourceProxy对数据源进行代理。数据源对象被代理后就具有了事务上下文的感知的能力。



不同持久化技术对应的事务管理器实现类

| org.springframework.orm.jpa.JpaTransactionManager            | 使用JPA进行持久化时，使用该事务管理器                        |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| org.springframework.orm.hibernate3.HibernateTransactionManager | 使用Hibernate3.0版本进行持久化时，使用该事务管理器           |
| org.springframework.jdbc.datasource.DataSourceTransactionManager | 使用Spring JDBC或iBatis等基于DataSource数据源的持久化技术时，使用该事务管理器 |
| org.springframework.orm.jdo.JdoTransactionManager            | 使用JDO进行持久化时，使用该事务管理器                        |
| org.springframework.transaction.jta.JtaTransactionManager    | 具有多个数据源的全局事务使用该事务管理器（不管采用何种持久化技术），如果希望在Java EE容器里使用JTA，我们将通过JNDI和Spring的JtaTransactionManager获取一个容器管理的DataSource。 |

大致来说，Spring支持两类事务，一种是本地连接事务（使用DataSourceTransactionManager），一种是JTA事务（使用JtaTransactionManager）。

JTA事务实现相对较好理解，在执行实际类的符合模式的方法时，代理类通过在连接点前后插入预处理过程（开始事务）和后处理过程（commit或rollbak）即可。因为JTA事务支持两阶段提交所以在代码中启动多少个连接（不同的connection）都能保证事务最终提交或者回滚。

作为基于DataSource的事务管理，实际上数据库连接Connection是最为核心的资源，事务的管理控制最终都会由Connection的相关方法来完成。关于其实现原理，不做深入探究。



事务传播行为类型

| PROPAGATION_REQUIRED      | 如果当前没有事务，就新建一个事务，如果已经存在一个事务，加入到这个事务中。这是最常见的选择。 |      |
| ------------------------- | ------------------------------------------------------------ | ---- |
| PROPAGATION_SUPPORTS      | 支持当前事务，如果当前没有事务，就以非事务方式执行。         |      |
| PROPAGATION_MANDATORY     | 使用当前的事务，如果当前没有事务，就抛出异常。               |      |
| PROPAGATION_REQUIRES_NEW  | 新建事务，如果当前存在事务，把当前事务挂起。                 |      |
| PROPAGATION_NOT_SUPPORTED | 以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。   |      |
| PROPAGATION_NEVER         | 以非事务方式执行，如果当前存在事务，则抛出异常。             |      |
| PROPAGATION_NESTED        | 如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行与PROPAGATION_REQUIRED类似的操作。 |      |

### ApplicationContext.xml里面配置示例

```
<tx:annotation-driven transaction-manager="transactionManager" proxy-target-class="true" />
```

以上是用@Transactional注解方式需要的配置节点，如果不用@Transactional注解方式配置事务，也可以用XML表达式配置事务，如下：

```
<aop:config expose-proxy="true">
    <!-- 只对业务逻辑层实施事务 -->
    <aop:pointcut id="txPointcut" expression="execution(* com.yougou.pc.api.service..*.*(..))"/>
    <aop:advisor advice-ref="txAdvice" pointcut-ref="txPointcut"/>
</aop:config>
<tx:advice id="txAdvice" transaction-manager="transactionManager">
<tx:attributes>
    <tx:method name="add*" propagation="REQUIRED"/>
    <tx:method name="del*" propagation="REQUIRED"/>
    <tx:method name="xx*" propagation="REQUIRED"/>
    <tx:method name="put*" read-only="true"/>
    <tx:method name="query*" read-only="true"/>
    <tx:method name="use*" read-only="true"/>
    <tx:method name="get*" read-only="true"/>
    <tx:method name="xx*" read-only="true"/>
     <tx:method name="*" propagation="REQUIRED"/>
</tx:attributes>
</tx:advice>
```

```
<!--iBatis的事务配置-->
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
 <property name="dataSource" ref="dataSourcejdbc"/>
</bean>
```

```
<!--JTA的事务配置--> 
<bean id="transactionManager" class="org.springframework.transaction.jta.JtaTransactionManager">
</bean>
```

```
<!-- 强烈建议用JdbcTemplate代替JdbcUtils -->
<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
<property name="dataSource" ref="dataSourcejdbc" />
</bean>
```

### 使用注解配置声明式事务：@Transactional



使用基于@Transactional注解的配置和基于XML的配置方式一样，它拥有一组普适性很强的默认事务属性，我们往往可以直接使用这些默认的属性就可以了，默认值如下：

| 事务传播行为 | PROPAGATION_REQUIRED                                         |
| ------------ | ------------------------------------------------------------ |
| 事务隔离级别 | SOLATION_DEFAULT，表示采用数据库本身的隔离级别               |
| 读写事务属性 | 读写事务                                                     |
| 超时时间     | 依赖于底层的事务系统的默认值                                 |
| 回滚设置     | 任何运行期异常引发回滚(unchecked)，任何检查型异常不会引发回滚 |

**@Transactional注解属性说明：**

| propagation            | 事务传播行为，通过以下枚举类提供合法值：org.springframework.transaction.annotation.Propagation，例如：@Transactional(propagation=Propagation.REQUIRES_NEW) |
| ---------------------- | ------------------------------------------------------------ |
| isolation              | 事务隔离级别，通过以下枚举类提供合法值：org.springframework.transaction.annotation.Isolation，例如：@Transactional(isolation=Isolation.READ_COMMITTED) |
| readOnly               | 事务读写性，boolean型，例如：@Transactional(readOnly=true)   |
| timeout                | 超时时间，int型，例如：@Transactional(timeout=10)            |
| rollbackFor            | 一组异常类，遇到时进行回滚，类型为：Class<? Extends Throwable>[]，默认为{}。例如：@Transactional(rollbackFor={SQLException.class})，多个异常之间可用逗号分隔。 |
| rollbackForClassName   | 一组异常类名，遇到时进行回滚，类型为String[]，默认值为{}。例如：@Transactional(rollbackForClassName={Exception"}) |
| noRollbackFor          | 和rollbackFor相对。                                          |
| noRollbackForClassName | 和rollbackForClassName相对。                                 |

@Transactional注解可以被应用于接口定义和接口方法、类定义和类的public方法上。

Spring建议在具体业务类上使用@Transactional注解。

方法处的注解会覆盖类定义处的注解，可以使用不同的事务管理器，例如：@Transactional("forum")，使用名为forum的事务管理器，需要在bean里面增加节点<qualifier value="forum"/>，如果不指定“限定符”，将默认使用“transationManager”命名对应的事务管理器。



指示spring事务管理器回滚一个事务的方法是在当前事务的上下文内抛出异常。spring事务管理器会捕捉任何未处理的异常，然后依据规则决定是否回滚抛出异常的事务。

默认配置下，spring只有在抛出的异常为运行时unchecked异常时才回滚该事务，也就是抛出的异常为RuntimeException的子类(Errors也会导致事务回滚)，而抛出checked异常则不会导致事务回滚。

可以明确的配置在抛出哪些异常时回滚事务，包括checked异常。也可以明确定义那些异常抛出时不回滚事务。

如果在@Transactional标记指定的方法里面，使用catch把异常吃掉了，那么这个事务是不会回滚的。

对应Spring+MyBatis事务，必须保证applicationContext.xml配置文件里面，transactionManger和Mybatis的数据源是一致的，如果不一致，则@Transactional注解的方法是没有事务的；可以用Corba作为数据源，是可以保证事务正常执行的，这个架构组做过测试。



演示一个老式的编程式事务管理，也感觉一下注解方式的优越：）

```
DefaultTransactionDefinition def = new DefaultTransactionDefinition();
def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
TransactionStatus status = txManager.getTransaction(def);
try {
 b.setBrandName("test");
 brandMapper.upddateBrand(b);           
 b.setBrandName("..."); //设置名字超长，让报错
 brandMapper.upddateBrand(b);
}
catch (Exception ex) {
 txManager.rollback(status);           
}
txManager.commit(status)
```

## 不同数据源下MyBatis配置数据源的办法

```
<!-- 会自动将basePackage中配置的包路径下的所有带有@Mapper标注的接口生成代理类，实现数据访问 -->
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
<!-- 多个数据源，这是mybatis-spring 1.2.0 的配置办法 -->
<!-- <property name="sqlSessionFactoryBeanName" value="sqlSessionFactoryXX" /> -->
<!-- 多个数据源，这是mybatis-spring 1.0.2 的配置办法 -->
<property name="sqlSessionFactory" ref="sqlSessionFactoryXX" />
<property name="sqlSessionTemplate" ref="sqlSessionTemplateXX"/>
<property name="basePackage" value="com/xx/xx" />
</bean>
```



# 后记

