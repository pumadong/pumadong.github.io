---

layout: single
title: Case Study（记一次TcpListenOverflows报警解决过程）
permalink: /case/tcp-listen-over-flows.html

classes: wide

author: Bob Dong

---

# 前言

这是刚进入公司1个月时候记录及处理的Case，记录在CSDN，[link](https://blog.csdn.net/puma_dong/article/details/46669499)。

后来在7年的工作过程中，整理记录很多Case，由于公司要求的原因，都发布在公司内部博客。

# 问题描述

2015-06-25，晚上21:33收到报警，截图如下：

![tcp-listen-over-flows](images/java-tcp-listen-over-flows-1.png)

此时，登陆服务器，用curl检查，发现服务报500错误，不能正常提供服务。

# 问题处理

```
tail各种日志，jstat看GC，不能很快定位问题，于是dump内存和线程stack后重启应用。

jps -v，找出Process ID

jstack -l PID > 22-31.log

jmap -dump:format=b,file=22-29.bin PID
```

## TcpListenOverflows


应用处理网络请求的能力，由两个因素决定：

1、应用的OPS容量（本例中是 就是我们的jetty应用：controller和thrift的处理能力）

2、Socket等待队列的长度（这个是os级别的，cat  /proc/sys/net/core/somaxconn 可以查看，默认是128，可以调优成了4192，有的公司会搞成32768）

当这两个容量都满了的时候，应用就不能正常提供服务了，TcpListenOverflows就开始计数，zabbix监控设定了>5发警报，于是就收到报警短信和邮件了。

这个场景下，如果我们到服务器上看看 listen情况，watch "netstat -s | grep listen"，会看到“xxx times the listen queue of a socket overflowed”，并且这个xxx在不断增加，这个xxx就是我们没有对网络请求正常处理的次数。

**参考文章：**

[关于tcp listen queue的一点事](http://www.douban.com/note/178129553/)

[如何判断是否丢掉用户请求](http://blog.sina.com.cn/s/blog_5374d6e30101lex3.html)

[linux里的backlog详解](http://blog.csdn.net/raintungli/article/details/37913765)

[linux下socket函数之listen的参数backlog](http://www.2cto.com/os/201207/139524.html)

[TCP SNMP counters](http://blog.chinaunix.net/uid-20695170-id-3073945.html)

[LINUX下解决netstat查看TIME_WAIT状态过多问题](http://www.itokit.com/2012/0516/73950.html)

理解了以上，我们已经可以大致认为，问题的根源，就是应用处理能力不足。以下的问题分析步骤，可以继续对此进行佐证。

# 问题分析

## 线程栈

首先看线程栈，大约12000多个线程，大量线程被TIME_WAIT/WAIT在不同的地址，偶有多个线程被同一个地址WAIT的情况，但是都找不到这个地址运行的是什么程序，貌似这个线程栈意义不大。

关于这点，还请同事一起进一步帮助分析，能否可以通过这个文件直接定位问题。

## Eclipse Memory Analyzer

MAT分析工具，分析JVM内存dump文件，下载地址： <http://www.eclipse.org/mat/downloads.php >。

通过分析，我们可以看到，内存中最多的类，是socket相关的，截图如下：

![tcp-listen-over-flows](images/java-tcp-listen-over-flows-2.png)

[Shallow heap & Retained heap](http://bjyzxxds.iteye.com/blog/1532937)

## Zabbix监控

![tcp-listen-over-flows](images/java-tcp-listen-over-flows-3.png)

# 问题解决


1、申请两台新虚拟机，挂上负载。

2、Jetty调优，增大线程数，maxThreads设置为500。

3、调用外部接口Timeout时间，统一调整为3秒，3秒前端就会超时，继续让用户走别的，所以我们的后端进程继续处理已经毫无意义。

# 根因分析

找个时间，分析日志，发现一个线程数太多的问题，两类线程太多（HttpClientPool、HttpMonitorCheckTimer ）；看事发时zabbix的截图，也是JVM线程很大，2万多了，并且不会减小。

![tcp-listen-over-flows](images/java-tcp-listen-over-flows-4.png)

定位到一个问题，每次都会news一个HttpClientHelper，new的过程中有两件事：

1、起一个HttpMonitor线程，里面有定时任务，所以这个线程是不死的。这个线程起名了，“Executors.newScheduledThreadPool(1, new NamedThreadFactory("HttpMonitorCheckTimer"))”，所以我们看到很多叫这个名字的线程

2、起一个HttpClientPool$IdleConnectionMonitorThread线程，这个线程是定时回收池子中过期以及空闲超过一点时间的线程，这个线程没起名，所以叫Thread-XXX这样的名字

![tcp-listen-over-flows](images/java-tcp-listen-over-flows-5.png)

```
grep HttpClientPool 22-31.log | wc -l     11857

grep HttpMonitorCheckTimer 22-31.log | wc -l    11856
```

这两种线程一定是1:1的关系，都是不死的。看起来Thread-XXX的多，这是因为其他的线程，没起名字的，也是这样命名的。所以运行Thread，还是起个名字吧。

查阅一些“PoolingHttpClientConnectionManager”线程池方式进行http调用的例子，根据HttpClientHelper的实现方式，写个PoolingHttpClientConnectionManagerDemo.java代码，运行1个小时，基本确认单例方式能正常运行；

于是修改单例方式调用HttpClientHelper，增加一点对调用次数的统计，上线。

# 后记

一次很好的学习过程，优先解决可用性问题，快速扩容。

然后定位到根因，根本上避免了问题再一次出现。

比较好的Code Review实践，对于一开始就发现这个问题也是很重要。