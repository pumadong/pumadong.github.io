---
layout: post
title: "Tomcat 性能优化指南"
date: 2025-12-16 18:30:00 +0800  # 标准格式
description: "优化 Tomcat 性能是一个系统工程，通常需要从**内存分配（JVM）**、**线程模型（Connector）以及操作系统**三个层面入手。"
---

优化 Tomcat 性能是一个系统工程，通常需要从**内存分配（JVM）**、**线程模型（Connector）以及操作系统**三个层面入手。

以下是针对生产环境最有效的参数优化建议：

------

## 1. JVM 内存优化 (`setenv.sh`)

Tomcat 运行在 JVM 之上，合理的内存配置能有效减少 GC（垃圾回收）频率，避免应用卡顿。

- **堆内存设置**：建议将 `-Xms` (初始堆) 和 `-Xmx` (最大堆) 设为一致，防止垃圾回收后重新分配堆大小。
- **元空间**：`-XX:MetaspaceSize` 避免频繁触发 Full GC。

**典型配置示例：**

Bash

```
# 在 bin/setenv.sh 中添加
export CATALINA_OPTS="$CATALINA_OPTS -server -Xms4g -Xmx4g -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m -XX:+UseG1GC"
```

> **提示：** 使用 **G1 收集器** (`-XX:+UseG1GC`) 是目前主流的选择，它在处理大内存时能提供更短的停顿时间。

------

## 2. 线程池优化 (Connector 核心参数)

在 `conf/server.xml` 中，针对 `<Connector>` 标签进行调整。这是提升并发处理能力的关键。

| **参数**              | **建议值** | **说明**                                                     |
| --------------------- | ---------- | ------------------------------------------------------------ |
| **maxThreads**        | 500 ~ 1000 | 最大线程数。根据 CPU 核心数和业务耗时调整，不是越大越好。    |
| **minSpareThreads**   | 50 ~ 100   | 最小备用线程数，即空闲时保持的线程量。                       |
| **acceptCount**       | 1000       | 当所有线程都在忙时，操作系统队列可存放的请求数。             |
| **maxConnections**    | 10000      | 服务器在任何给定时间接受和处理的最大连接数（NIO 模式下可设置较高）。 |
| **connectionTimeout** | 20000      | 等待连接超时的毫秒数。                                       |

**配置示例：**

XML

```
<Connector port="8080" protocol="org.apache.coyote.http11.Http11NioProtocol"
           connectionTimeout="20000"
           redirectPort="8443"
           maxThreads="800"
           minSpareThreads="100"
           acceptCount="1000"
           maxConnections="10000"
           enableLookups="false"
           URIEncoding="UTF-8" />
```

------

## 3. 运行模式选择 (IO 模型)

Tomcat 支持三种运行模式，建议根据版本选择：

- **NIO (Non-blocking IO)**：Tomcat 8/9 的默认模式，适合一般的长连接和高并发。
- **NIO2 (Asynchronous IO)**：Tomcat 8.5+ 支持，利用操作系统的异步机制。
- **APR (Apache Portable Runtime)**：性能最强，但需要安装本地库（openssl, apr 等），配置较复杂。

> **优化建议：** 如果不想折腾复杂的库依赖，使用 `org.apache.coyote.http11.Http11Nio2Protocol` 通常能获得极佳的性能。

------

## 4. 其他细节优化

### 禁用 DNS 查询

在 Connector 中设置 `enableLookups="false"`。如果设为 true，Tomcat 会尝试通过 IP 反查域名，这会显著增加请求延迟。

### 开启 GZIP 压缩

对于返回大量文本（JSON, HTML）的接口，开启压缩可以节省带宽，提升加载速度。

XML

```
compression="on"
compressionMinSize="2048"
compressableMimeType="text/html,text/xml,text/plain,application/json,application/javascript"
```

### 操作系统限制

在 Linux 下，务必调高文件句柄数限制（ulimit），否则高并发下会报错 `Too many open files`。

Bash

```
# /etc/security/limits.conf
* soft nofile 65535
* hard nofile 65535
```

------

## 总结建议

1. **先监控再调优**：使用 `jvisualvm` 或 `Prometheus + Grafana` 观察当前的线程占用和 GC 情况。
2. **压测验证**：每次修改参数后，使用 JMeter 进行压力测试，观察吞吐量（TPS）和响应时间的变化。