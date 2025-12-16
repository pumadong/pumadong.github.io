---
layout: single
title: 本机环境
permalink: /local-environment.html

classes: wide

author: Bob Dong


---

# 前言

配置一些本机的常用环境。

# 待解决

- redis的配置文件中如何设置变量：logfile redis-${port}.log这种。
- Idea 2024 双击文件经常不能打开。
- Sentinel节点如何相互发现，配置之后如何相互发现。
- 为什么redis cluster是对16383取模，而不是16384.

# Redis

**RDB和AOF比较**

​							RDB 	AOF
启动优先级：	低		高
体积：		小		大
回复速度：	快		慢
数据安全性：	丢数据	根据策略决定
轻重：		重		轻

**RDB最最佳策略**：

- 建议关掉，注意：但是主从全量复制是会自动触发的。

**AOF最佳策略**：

- 建议开，默认宕机可能丢1秒数据。只做缓存可以关闭掉。
- AOF重写集中管理。
- everysec

**最佳策略：**

- 小分片，maxMemory最大内存4个G，fork或者rdb会较小开销。
- 监控（硬盘、内存、负载、网络）。
- 足够的内存。

**开发运维常见问题：**

- fork操作：
  - 问题：同步操作；与内存量息息相关，内存越大，耗时越长（与机器类型相关）；info: lastest_fork_usec执行时间。
  - 改善：优先使用物理机或者高效支持fork操作的虚拟化技术；控制Redis实例的最大可用内存：maxMemory；合理配置Linux内存分片策略：vm.overcommit_memory=1（默认0，代表内存不足就不分配）；降低fork频率：例如放宽AOF重写自动触发时机，不必要的全量复制。
- 进程外开销
  - CPU开销：RDB和AOF文件生成，属于CPU密集型；优化：不做CPU绑定，不和CPU密集型应用一起部署。
  - 内存开销：fork内存开销，copy-on-write；优化：e'cho never > /sys/kernel/mm/transparent_hugepage/enabled
  - 硬盘开销：AOF和RDB文件写入，可以结合iostat，iotop分析；优化：不要he高硬盘负载服务部署一起，no-append-fsync-on-rewrite = yes，根据写入量决定磁盘类型，例如ssd，单机多实例持久化文件目录可以考虑分盘。
- AOF追加阻塞
  - 主线程对比上次fsync时间，如果大于2秒，会阻塞，小于2秒通过（丢2秒数据）。
  - 如果发生AOF阻塞，redis日志会有：Asynchronous AOF fsync is taking too long(disk is busy?). 执行命令info persistence，会记录发生delay的次数，例如，oaf_delayed_fsync: 100。
- 单机多实例部署

# 主从复制

命令：slaveof ip port，无需重启，不便于管理

配置文件：需要重启，便于管理

slaveof ip port

slave-read-only yes

info replication 查看主从相关信息。

**全量复制开销：**

- bgsave时间
- RDB文件网络传输时间
- 从节点清空数据时间（flushall）
- 从节点加载RDB的时间
- 可能的AOF重写时间

# 网络

RHEL7.9虚拟主机IP地址：192.168.1.200

Redis端口：6382，本机链接Redis，需要关闭Server的防火墙或者打开6382端口，同时，本地需要把这个IP加入VPN的ByPass中。

在redhat7/centos7中，Linux默认的防火墙是firewalld，启动与关闭方式如下：

```
#查看防火墙状态
firewall-cmd --state 
systemctl status firewalld

#临时开启，重启机器后失效 
systemctl unmask firewalld 
systemctl start firewalld 

#临时关闭，重启机器后失效 
systemctl stop firewalld

#开机自启动 
systemctl enable firewalld

#关闭开机自启动
systemctl disable firewalld


# 要确保通过访问firewalld D-Bus接口以及如果需要其他服务也未
# 启动firewalld firewalld，使用root输入以下命令：
systemctl mask firewalld
```