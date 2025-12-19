---
layout: post
title: "Docker From Zero To Hero"
---

# Docker架构图

![Docker架构图](https://user-images.githubusercontent.com/43399466/217507877-212d3a60-143a-4a1d-ab79-4bb615cb4622.png)

### 总结：它是如何工作的？

当你输入 `docker run hello-world` 时：

1. **Docker CLI** 接收命令并将其转换为 **REST API** 调用。
2. **Docker Daemon** 接收到请求，检查本地是否有镜像。
3. 如果本地没有，Daemon 会从 **Docker Registry** 拉取。
4. 最后，Daemon 调用底层驱动创建并启动容器。

### 演示环境准备

1. AWS，申请一个2核CPU、1G内存，安装ubuntu最新版本的EC2主机。Inbound Rules开放8080端口。

   例如：`ssh -i "bob.pem" ubuntu@ec2-34-224-28-231.compute-1.amazonaws.com`

2. https://hub.docker.com/，申请账户。

   这是公用的Docker Image注册中心，Image->Registry，类似于Source Code -> GitHub

# 安装Docker

1. 安装

   官方地址：https://docs.docker.com/get-started/get-docker/。
   
   使用我们之前准备的ubuntu主机安装：
   
    ```
    sudo apt update
    sudo apt install docker.io -y
    ```


2. 验证

   ```
   sudo systemctl status docker
   ```

   如果发现没有启动，使用如下命令启动：

   ```
   sudo systemctl start docker
   ```

3. 给当前登录用户赋予运行Docker命令的权限

   ```
   sudo usermod -aG docker ubuntu
   ```

   为什么需要赋权：

   是因为Docker默认是需要用root权限安装的，这会带来安全问题，后续会讲如何解决这个问题。

4. 使用Docker命令执行一个hello-word镜像

   ```
   docker run hello-world
   ```

   可以看到输出如下，代表Docker命令可以正常执行了。

   ```
   ....
   ....
   Hello from Docker!
   This message shows that your installation appears to be working correctly.
   ...
   ...
   ```

# 演示2个从GubHub源码构建Docker的例子

## 第1个例子：Python写的HelloWorld

1. 从GitHub克隆源代码

   ```
   git clone https://github.com/pumadong/docker-python-hello-world.git
   cd  docker-python-hello-world
   ```

2. Dockerfile

   ```
   FROM ubuntu:latest
   
   # Set the working directory in the image
   WORKDIR /app
   
   # Copy the files from the host file system to the image file system
   COPY . /app
   
   # Install the necessary packages
   RUN apt-get update && apt-get install -y python3 python3-pip
   
   # Set environment variables
   ENV NAME World
   
   # Run a command to start the application
   CMD ["python3", "app.py"]
   ```

   这种方式构建的镜像文件很大，约500M，实际都是多阶段构建Distroless 镜像，也就是没有Linux发行版的镜像。

3. 构建容器

   ```
   docker build -t bob0516/docker-python-hello-world:latest .
   ```

   验证容器被正确生成：

   ```
   docker images
   ```

4. 运行容器

   ```
   docker run -it bob0516/docker-python-hello-world
   ```

5. 推送image到注册中心

   登录注册中心，本例中是hub.docker.com这个公用注册中心。

   ```
   docker login
   ```

   推送image：

   ```
   docker push bob0516/docker-python-hello-world
   ```

   可以看到image已经推送成功：

   https://hub.docker.com/r/bob0516/docker-python-hello-world

## 第2个例子：多阶段构建Java写的Web应用

1. Clone Source Code

   ```
   git clone https://github.com/pumadong/docker-java-web-app.git
   cd docker-java-web-app
   ```

2. Dockfile

   ```
   # --- 第一阶段：构建阶段 ---
   FROM maven:3.8.6-jdk-8-slim AS build
   
   # 设置工作目录
   WORKDIR /app
   
   # 1. 优化缓存：先拷贝 pom.xml 并下载依赖
   # 这样只要 pom.xml 没变，后续构建就可以跳过依赖下载步骤
   COPY pom.xml .
   RUN mvn dependency:go-offline
   
   # 2. 拷贝源代码并打包
   COPY src ./src
   RUN mvn clean package -DskipTests
   
   # --- 第二阶段：运行阶段 ---
   # FROM eclipse-temurin:8-jre-alpine
   FROM gcr.io/distroless/java:8
   
   # 设置工作目录
   WORKDIR /app
   
   # 从构建阶段拷贝生成的 jar 包到当前镜像
   # 请根据你 pom.xml 中的 artifactId 和 version 修改 jar 包名称
   COPY --from=build /app/target/*.jar docker-java-web-app-0.0.1-SNAPSHOT.jar
   
   # 暴露项目端口（假设为 8080）
   # EXPOSE 并不实际开放端口，它更像是一种文档说明，告诉开发者或系统该应用预期监听哪个端口
   EXPOSE 8080
   
   # 启动命令
   # 通过环境变量指定 Spring Profile
   ENTRYPOINT ["java","-Dspring.profiles.active=prod", "-jar", "docker-java-web-app-0.0.1-SNAPSHOT.jar"]
   ```

   这个Dockfile既是一个Java应用，也是一个多阶段构建的例子

3. Build

   ```
   docker build -t bob0516/docker-java-web-app:latest .
   ```

4. Run

   ```
   docker run -d -p 8080:8080 -v /data/applogs:/app/logs bob0516/docker-java-web-app:latest
   ```

   把镜像里面的日志输出路径，挂载到宿主机的/data/applogs路径下，这样容器重启，日志也不会丢失。

5. Visit by browserDockfile

   访问http://公网IP:8080，站点正常显示。

## Java8 Dockerfile最佳实践

### 1. 选择合适的镜像基座 (Base Image)

不要直接使用 `FROM java:8` 或 `FROM openjdk:8`，因为它们通常基于较大的 Debian 镜像，且早已停止维护。

- **推荐使用 Eclipse Temurin (原 AdoptOpenJDK):** 它是目前社区维护最活跃、最可靠的发行版。
- **选择 Alpine 版本:** 如果追求极小体积。
- **选择 Slim 版本:** 如果 Alpine 环境下某些 C 库（如 glibc）兼容性有问题。

Dockerfile

```
# 推荐：体积小、安全且有持续维护
FROM eclipse-temurin:8-jre-alpine
```

------

### 2. 多阶段构建 (Multi-stage Builds)

这是减小镜像体积最有效的手段。在编译阶段使用完整的 JDK，在运行阶段仅使用精简的 JRE。

Dockerfile

```
# 第一阶段：编译 (Build Stage)
FROM maven:3.8.6-openjdk-8 AS build
WORKDIR /app
COPY pom.xml .
# 预先下载依赖，利用缓存
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

# 第二阶段：运行 (Run Stage)
FROM eclipse-temurin:8-jre-alpine
WORKDIR /app
# 只从编译阶段拷贝最终生成的 jar 包
COPY --from=build /app/target/*.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

------

### 3. 解决 Java 8 的容器感知问题 (Cgroup)

Java 8 的早期版本（8u131 之前）无法识别 Docker 限制的内存和 CPU。如果不对 JVM 进行配置，它可能会尝试使用宿主机的全部内存，从而导致 **OOMKilled**。

- **最佳实践：** 确保使用 **8u191** 或更高版本，并开启 `UseContainerSupport`。

Dockerfile

```
# 在 ENTRYPOINT 中加入相关参数
ENTRYPOINT ["java", "-XX:+UnlockExperimentalVMOptions", "-XX:+UseCGroupMemoryLimitForHeap", "-XX:MaxRAMFraction=2", "-XshowSettings:vm", "-jar", "app.jar"]
```

*注：对于 8u191 之后的版本，推荐使用 `-XX:+UseContainerSupport`。*

------

### 4. 以非 Root 用户运行

出于安全考虑，容器内的程序不应以 root 权限运行。

Dockerfile

```
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

COPY --from=build --chown=spring:spring /app/target/*.jar app.jar
```

------

### 5. 优化镜像层缓存

将不经常变动的指令放在前面。例如，先拷贝 `pom.xml` 下载依赖，再拷贝源码，这样在源码改动时，依赖层可以被复用，极大加快构建速度。

------

### 6. 完整的最佳实践示例

这是一个综合了安全、体积和性能的 Java 8 Dockerfile 模板：

Dockerfile

```
# Stage 1: Build
FROM maven:3.8-eclipse-temurin-8-alpine AS build
WORKDIR /build
# 缓存 Maven 依赖
COPY pom.xml .
RUN mvn dependency:go-offline

COPY src ./src
RUN mvn package -DskipTests

# Stage 2: Runtime
FROM eclipse-temurin:8-jre-alpine
LABEL maintainer="yourname@example.com"

# 设置时区
RUN apk add --no-cache tzdata && \
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone

# 创建非 root 用户
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /home/appuser

# 从 Build 阶段拷贝 Jar
COPY --from=build --chown=appuser:appgroup /build/target/*.jar app.jar

USER appuser

# 优化 JVM 参数：容器感知、内存限制、GC 优化
ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -XshowSettings:vm -Djava.security.egd=file:/dev/./urandom"

EXPOSE 8080

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

------

### 关键点总结

| **优化项**   | **建议**                                 |
| ------------ | ---------------------------------------- |
| **基础镜像** | 使用 `eclipse-temurin:8-jre-alpine`      |
| **体积控制** | 必须使用 **多阶段构建** (Multi-stage)    |
| **内存安全** | 开启 `-XX:+UseContainerSupport`          |
| **安全加固** | 使用 `USER` 指令切换非 root 用户         |
| **时区处理** | 明确安装 `tzdata` 并设置 `Asia/Shanghai` |

# FAQ

# 一、Docker是什么？

简单来说，**Docker 是一个开源的应用容器引擎**。它让开发者可以把应用程序及其所有的依赖项（代码、库、配置文件等）打包进一个标准化的单位中，这个单位就叫**容器（Container）**。

你可以把 Docker 想象成货运业的**集装箱**。

### 1. 为什么要用 Docker？（核心痛点）

在 Docker 出现之前，程序员经常遇到一个经典问题：“这程序在我的电脑上跑得好好的，怎么到你的服务器上就报错了？”

这是因为两台电脑的操作系统版本、环境变量、软件库版本不一致导致的。Docker 通过将应用和环境“整捆打包”，确保了：

- **环境一致性：** 无论是在开发环境、测试环境还是生产环境，跑的都是同一个容器。
- **轻量化：** 相比于传统的虚拟机（VM），容器不需要运行完整的操作系统内核，启动速度以秒计，且占用资源极低。

------

### 2. Docker 的核心概念

要理解 Docker，只需记住这三个核心名词：

- **镜像 (Image)：** 相当于一个“模板”或“安装包”。它是一个只读的文件，里面包含了运行软件所需的所有东西。
- **容器 (Container)：** 镜像是静态的，容器是动态的。镜像是类，容器就是实例。你可以通过一个镜像启动多个容器。
- **仓库 (Registry)：** 存放镜像的地方。最著名的就是官方的 Docker Hub。

------

### 3. Docker vs 虚拟机 (VM)

这是最常见的对比。它们的主要区别在于**隔离的层级**：

| **特性**     | **虚拟机 (VM)**             | **Docker 容器**      |
| ------------ | --------------------------- | -------------------- |
| **操作系统** | 每个 VM 都运行一个完整的 OS | 共享宿主机的 OS 内核 |
| **性能**     | 占用大量内存和磁盘，启动慢  | 极其轻量，秒级启动   |
| **隔离性**   | 彻底的硬件级隔离（更安全）  | 进程级隔离           |

------

### 4. 总结：Docker 能做什么？

1. **快速部署：** 几行命令就能搭建起复杂的数据库或 Web 服务器。
2. **微服务架构：** 将大型应用拆分成多个互相隔离的小容器运行。
3. **消除环境差异：** “一次构建，到处运行”。

# 二、Docker生命周期

Docker 容器的生命周期（Lifecycle）是指一个容器从被创建、运行、暂停、停止到最终被删除的完整过程。理解这一过程的关键在于掌握容器在不同阶段的状态转化。

------

### 1. 五个核心状态

Docker 容器通常会经历以下五个主要状态：

| **状态**             | **说明**                                                     | **对应命令**                  |
| -------------------- | ------------------------------------------------------------ | ----------------------------- |
| **Created (已创建)** | 容器已根据镜像创建，拥有自己的 ID，但**尚未启动**，不占用 CPU 和内存。 | `docker create`               |
| **Running (运行中)** | 容器的主进程正在运行，正在执行其定义的任务。                 | `docker start` / `docker run` |
| **Paused (暂停)**    | 容器内所有进程被挂起（通过 `SIGSTOP` 信号），**内存状态保留**，但不再消耗 CPU。 | `docker pause`                |
| **Stopped (已停止)** | 容器主进程已退出，不再运行，但其文件系统和元数据仍保留在磁盘上。 | `docker stop`                 |
| **Deleted (已删除)** | 容器的所有数据（包括可写层）从宿主机彻底移除。               | `docker rm`                   |

------

### 2. 生命周期关键路径

我们可以将这一过程简化为以下几个阶段：

#### A. 从镜像到运行

- **`docker create`**: 仅仅是准备好容器的文件系统和配置。
- **`docker start`**: 启动已经创建或停止的容器。
- **`docker run`**: 这是最常用的组合命令，它相当于先执行了 `create` 再执行 `start`。如果本地没有镜像，它还会先执行 `pull`。

#### B. 暂停与恢复

- 当你需要临时释放 CPU 资源（例如运行另一个高负载任务）但不希望丢失容器当前的内存状态时，可以使用 **`docker pause`**。
- 使用 **`docker unpause`** 可以瞬间恢复进程运行。

#### C. 停止与终止

- **`docker stop`**: “优雅”地停止。它先向进程发送 `SIGTERM` 信号，给进程一定时间处理清理工作（如保存数据、关闭连接），如果超时则强制发送 `SIGKILL`。
- **`docker kill`**: “暴力”地停止。直接发送 `SIGKILL` 信号，立即终止进程，可能导致数据丢失。

#### D. 清理

- **`docker rm`**: 容器停止后，如果不手动删除，它会一直占用磁盘空间。使用 `docker rm` 彻底销毁它。
- **技巧**: 在 `docker run` 时加上 `--rm` 参数，可以在容器停止后自动将其删除，非常适合临时任务。

------

### 3. 常见状态转化图示 (逻辑流程)

1. **Image** ➔ `create` ➔ **Created**
2. **Created** ➔ `start` ➔ **Running**
3. **Running** ➔ `pause` ➔ **Paused** ➔ `unpause` ➔ **Running**
4. **Running** ➔ `stop` ➔ **Stopped**
5. **Stopped** ➔ `rm` ➔ **Destroyed**

# 三、Docker核心组件

Docker 的核心架构采用的是 **客户端-服务器（C/S）架构**。简单来说，它由不同的组件协作，完成从“编写代码”到“容器运行”的全过程。

为了让你快速上手，我们可以将 Docker 的组件分为以下几个核心部分：

------

## 1. Docker 核心三大件 (Core Objects)

这是你日常使用 Docker 时接触频率最高的部分：

- **镜像 (Image):** 镜像是一个**只读的模板**，包含了运行应用程序所需的所有代码、运行环境、库和配置文件。你可以把它类比为安装操作系统的 ISO 文件或虚拟机的镜像。
- **容器 (Container):** 容器是镜像的**运行实例**。你可以启动、停止、删除镜像。容器之间是相互隔离的，非常轻量。如果说镜像是“类”，那么容器就是“对象”。
- **仓库 (Registry):** 用来**存储和分发镜像**的地方。最著名的公开仓库是 [Docker Hub](https://hub.docker.com/)。企业内部通常也会搭建私有仓库（如 Harbor）。

------

## 2. Docker 后端架构 (The Engine)

### **Docker Daemon (dockerd)**

它是 Docker 的“大脑”，以守护进程的形式运行在后台。它负责管理所有的 Docker 对象，如镜像、容器、网络和存储卷。它通过监听 Docker API 请求来执行指令。

### **Docker Client (docker)**

这是我们最常用的命令行工具（CLI）。当你输入 `docker run` 时，客户端会将命令发送给 `dockerd`。客户端和守护进程可以在同一个系统上运行，也可以连接到远程的 Docker 守护进程。

------

## 3. 底层运行时组件 (Runtime)

在更深层次，Docker 并不是自己直接去操作 Linux 内核，而是通过以下组件实现的：

- **containerd:** 这是一个工业级的容器运行时，负责管理容器的生命周期（从拉取镜像到执行容器）。
- **runc:** 这是最底层的组件，遵循 OCI (Open Container Initiative) 标准。它的唯一工作就是与 Linux 内核交互，利用 **Namespaces**（实现环境隔离）和 **Cgroups**（实现资源限制）来真正创建一个容器。

------

## 4. 网络与存储 (Network & Storage)

- **Docker Networking:** 允许容器之间进行通信，或与外部网络通信。常见的模式有 `bridge`（桥接）、`host`（主机模式）和 `none`。
- **Docker Volumes:** 容器的文件系统通常是随删随消失的。为了持久化数据（比如数据库的路径），Docker 提供了 **Volumes（卷）**，将宿主机的目录挂载到容器内部。

------

## 总结：工作流示意

1. **Build:** 编写 `Dockerfile`，构建出 **Image**。
2. **Push:** 将 **Image** 推送到 **Registry**。
3. **Run:** 从仓库拉取镜像，由 **Daemon** 调用运行时创建并启动 **Container**。

# 四、Docker COPY 与 ADD 区别

在 Dockerfile 中，`COPY` 和 `ADD` 都用于将文件或目录从宿主机（或构建上下文）复制到镜像中。

**结论先行：** 在绝大多数情况下，应优先使用 **`COPY`**。只有在需要其特有的“自动解压”功能时，才考虑使用 `ADD`。

------

### 1. 核心区别对照表

| **特性**           | **COPY**                   | **ADD**                      |
| ------------------ | -------------------------- | ---------------------------- |
| **基本功能**       | 将本地文件/目录复制到镜像  | 将本地文件/目录复制到镜像    |
| **远程 URL 支持**  | ❌ 不支持                   | ✅ 支持（但不推荐）           |
| **压缩包自动解压** | ❌ 不支持（原样复制）       | ✅ 支持（仅限本地压缩包）     |
| **多阶段构建**     | ✅ 支持 `--from` 跨阶段复制 | ❌ 不支持                     |
| **透明度与安全性** | ⭐️ 高（行为单一、可预测）   | ⚠️ 低（行为复杂、有安全隐患） |

------

### 2. ADD 的特殊能力（及槽点）

`ADD` 指令比 `COPY` 多了两项“魔法”功能，但这两项功能在现代 Docker 实践中都有更好的替代方案：

- **自动解压本地压缩包：**
  - 如果你执行 `ADD example.tar.gz /app/`，Docker 会自动将其解压到目标目录。
  - **限制：** 这一功能只对**本地**压缩包有效。如果从 URL 下载的压缩包，它**不会**自动解压。
- **支持下载远程 URL：**
  - 你可以通过 `ADD http://example.com/file.txt /` 下载文件。
  - **为什么不推荐：** 这样做会产生一个额外的镜像层。**更优做法**是使用 `RUN curl` 或 `wget`，这样可以在同一层内下载、解压并删除安装包，从而减小镜像体积。

------

### 3. 为什么官方推荐 COPY？

1. **可预测性：** `COPY` 的功能非常纯粹，就是“搬运”。你不用担心它会突然把一个 `.tar.gz` 文件解压开来，导致镜像结构混乱。
2. **更轻量：** `COPY` 的处理逻辑简单，构建性能略优。
3. **支持多阶段构建：** 在多阶段构建（Multi-stage builds）中，你需要使用 `COPY --from=build /out/app .` 从之前的构建阶段提取产物，这是 `ADD` 无法做到的。

------

### 💡 最佳实践建议

- **默认使用 `COPY`：** 搬运代码、配置文件、静态资源时，始终用 `COPY`。
- **唯一使用 `ADD` 的场景：** 当你需要将本地的一个 `tar` 压缩包（如 rootfs）解压到镜像中时。
- **下载文件：** 尽量使用 `RUN wget` 或 `RUN curl`，并配合 `rm` 命令清理临时文件。

> **提示：** 无论使用哪种指令，被复制的文件路径必须在 **Docker 构建上下文（Build Context）** 之内（通常是 Dockerfile 所在的目录及其子目录）。

# 五、Docker Entrypoint 和 CMD的不同

简单来说，`ENTRYPOINT` 和 `CMD` 都是用来指定容器启动时运行的程序的，但它们的**灵活性**和**角色**不同。

------

## 核心区别

| **特性**     | **CMD**                                      | **ENTRYPOINT**                                           |
| ------------ | -------------------------------------------- | -------------------------------------------------------- |
| **主要定位** | 设置**默认**命令或参数。                     | 设置容器的**主程序**（不可轻易更改）。                   |
| **覆盖行为** | 很容易被 `docker run` 后的参数**完全替换**。 | 不会被 `docker run` 的参数替换，参数会**追加**到它后面。 |
| **强制性**   | 弱（建议作为可选默认项）。                   | 强（容器启动时必须执行的“入口”）。                       |
| **如何覆盖** | `docker run <image> <new_command>`           | 需要显式使用 `--entrypoint` 参数。                       |

------

## 它们的三种互动方式

### 1. 只使用 CMD

如果你希望容器像一个通用工具，用户可以随意运行不同的命令：

Dockerfile

```
FROM ubuntu
CMD ["echo", "Hello World"]
```

- 直接运行：`docker run my_image` $\rightarrow$ 输出 `Hello World`
- 带参数运行：`docker run my_image ls` $\rightarrow$ **不执行 echo**，而是运行 `ls`。

### 2. 只使用 ENTRYPOINT

如果你希望容器表现得像一个专门的“可执行程序”：

Dockerfile

```
FROM ubuntu
ENTRYPOINT ["echo", "Hello"]
```

- 带参数运行：`docker run my_image World` $\rightarrow$ 输出 `Hello World`（`World` 被当作参数传给了 `echo`）。

### 3. 组合使用（推荐做法）

将 `ENTRYPOINT` 设置为固定命令，`CMD` 设置为**默认参数**。

Dockerfile

```
FROM ubuntu
ENTRYPOINT ["echo"]
CMD ["Hello World"]
```

- `docker run my_image`  ➔ 输出 `Hello World`。
- `docker run my_image "Hi there"`  ➔ 输出 `Hi there`（这里 `Hi there` **覆盖了 CMD**，但 `ENTRYPOINT` 依然执行）。

------

## 注意事项：Exec 格式 vs Shell 格式

在写 Dockerfile 时，强烈建议使用 **JSON 数组格式**（即 `Exec` 格式）：

- **推荐 (Exec):** `ENTRYPOINT ["executable", "param1"]` —— 这种方式会让程序以 PID 1 运行，能正常接收 UNIX 信号（如 `SIGTERM`）。
- **不推荐 (Shell):** `ENTRYPOINT executable param1` —— 这种方式会启动一个 `/bin/sh -c` 子进程，可能会导致 `docker stop` 时程序无法优雅退出。

> **总结建议：** > * 如果你是在构建一个特定的应用镜像（如数据库、Web 服务），请使用 **`ENTRYPOINT`**。
>
> - 如果你想为用户提供默认的执行选项，请使用 **`CMD`**。
> - 最灵活的方案是：**`ENTRYPOINT` 定义程序，`CMD` 定义参数。**

# 六、Docker中的网络类型

在 Docker 中，网络（Networking）是容器之间以及容器与外部世界通信的核心。Docker 提供了一套灵活的驱动程序，允许你根据不同的应用场景选择最合适的网络类型。

以下是 Docker 的五种主要网络模式及其详细解析：

------

### 1. Bridge（桥接网络）

这是 Docker **默认**的网络类型。当你启动一个容器而不指定网络模式时，它就会连接到这个私有网桥。

- **工作原理：** Docker 会在主机上创建一个名为 `docker0` 的虚拟网桥。每个容器都会分配到一个私有 IP 地址。
- **适用场景：** 适用于在**同一个 Docker 宿主机**上运行的多个独立容器之间的通信。
- **特点：** 容器之间可以通过 IP 通信；如果使用自定义 Bridge 网络，还可以直接通过**容器名称**（DNS）进行通信。

```
docker run -d --name login nginx:latest
docker run -d --name logout nginx:latest

# 172.17.0.3
docker inspect login

# 更新软件包列表
sudo apt update
# 安装 iputils-ping 软件包
sudo apt install iputils-ping -y

# 172.17.0.4
docker inspect logout

docker exec -it login /bin/bash
ping 172.17.0.4
```

------

### 2. Host（主机网络）

在这种模式下，容器不会获得独立的网络命名空间，而是直接共享宿主机的网络栈。

- **工作原理：** 容器直接使用宿主机的 IP 和端口。例如，如果你在容器中运行一个监听 80 端口的服务，那么它直接占用宿主机的 80 端口。
- **适用场景：** 对**网络性能**要求极高的情况（减少了 NAT 转换的开销），或者需要容器处理大量端口的场景。
- **缺点：** 缺乏隔离性，容器与宿主机端口容易冲突。

------

### 3. Overlay（覆盖网络）

Overlay 网络用于连接**不同宿主机**上的 Docker 容器。

- **工作原理：** 它在多个 Docker 守护进程（Docker Swarm 集群）之间创建一个分布式网络。它利用 VxLAN 技术在底层物理网络之上封装了一层逻辑网络。
- **适用场景：** **微服务架构**、Docker Swarm 集群或多机部署。
- **特点：** 允许跨主机通信，无需担心底层路由配置。

------

### 4. Macvlan 网络

Macvlan 允许你为容器分配一个**物理网络上的 MAC 地址**，使其看起来像是一台真实的物理设备。

- **工作原理：** 容器直接连接到主机的物理网卡，并拥有在该网段内的独立 IP。
- **适用场景：** 需要容器直接出现在物理网络中（例如某些遗留系统监控、网络诊断工具），或者需要绕过 Docker 网桥的场景。
- **注意：** 这种模式比较复杂，通常需要宿主机网卡开启混杂模式。

------

### 5. None（无网络）

这种模式会将容器放入自己的网络栈中，但不进行任何网络配置。

- **工作原理：** 容器只有一个回环接口（`lo` / 127.0.0.1），没有外部网卡。
- **适用场景：** 极高安全要求的离线任务、批处理计算或只需要处理本地文件系统的容器。

------

### 核心对比表

| **网络类型** | **适用范围** | **隔离性** | **主要用途**                  |
| ------------ | ------------ | ---------- | ----------------------------- |
| **Bridge**   | 单机         | 高         | 标准容器化应用，默认首选      |
| **Host**     | 单机         | 无         | 高性能网络请求，消除 NAT 损耗 |
| **Overlay**  | 多机         | 高         | Swarm 集群，跨主机微服务通信  |
| **Macvlan**  | 单机/多机    | 极高       | 容器需要独立物理 IP 的场景    |
| **None**     | 无           | 完全隔离   | 运行不需要联网的安全任务      |

### 6.容器间的网络隔离机制

Docker 的隔离主要依靠 Linux 内核的两大技术：**Namespaces（命名空间）** 和 **Control Groups（控制组）**，在网络层面则具体表现为：

| **隔离手段**          | **描述**                                                     |
| --------------------- | ------------------------------------------------------------ |
| **Network Namespace** | 每个容器拥有独立的网络栈（网卡、路由表、防火墙规则），实现逻辑隔离。 |
| **Iptables 规则**     | Docker 通过操作宿主机的 `iptables` 来实现不同网络间的访问控制。 |
| **默认不互通**        | 位于不同“自定义网桥”上的容器，默认是无法直接 Ping 通的，必须通过 `docker network connect` 手动打通。 |
| **端口映射**          | 除非显式使用 `-p` 暴露端口，否则外部网络无法访问容器内部私有网络。 |

#### 隔离的实际表现：

1. **同网段可见**：连接到同一个自定义桥接网络（Bridge）的容器可以互相访问所有端口。
2. **跨网段隔离**：如果 Container A 在 `web-net`，Container B 在 `db-net`，即使在同一台机器上，它们也无法直接通信，除非你创建一个连接。

### 7.总结：该选哪种网络？

**本地开发/小型 Web 应用**：使用 **自定义 Bridge**。

**追求极致性能**：使用 **Host**。

**跨服务器分布式架构**：使用 **Overlay**。

**对安全性要求极高的内部处理**：使用 **None**。

### 8.自定义Bridge

在 Docker 中，使用默认的 `bridge` 网络虽然方便，但在生产或复杂开发环境中，**自定义桥接网络 (User-defined Bridge Network)** 是更好的选择。它提供了更好的隔离性、**自动 DNS 解析**（可以通过容器名直接通信）以及更灵活的配置。

以下是定制并使用自定义 Bridge 网络的步骤：

#### 1. 创建自定义 Bridge 网络

你可以通过 `docker network create` 命令来定制子网网段、网关等参数。

Bash

```
docker network create \
  --driver bridge \
  --subnet 192.168.10.0/24 \
  --gateway 192.168.10.1 \
  my_custom_network
```

- **--driver bridge**: 指定驱动类型为桥接。
- **--subnet**: 手动指定 IP 地址段（避免与宿主机或其他网络冲突）。
- **--gateway**: 指定该虚拟网段的网关。

------

#### 2. 运行容器并连接到网络

在启动容器时，使用 `--network` 参数将其加入你创建的网络。

- **启动容器 A (Web 服务):**

  Bash

  ```
  docker run -d --name web_server --network my_custom_network nginx
  ```

- **启动容器 B (测试工具):**

  Bash

  ```
  docker run -it --name test_client --network my_custom_network alpine sh
  ```

------

#### 3. 验证网络特性 (DNS 解析)

自定义网络最强大的地方在于**内置 DNS**。在 `test_client` 容器内部，你可以直接通过容器名访问 `web_server`，而不需要知道它的 IP 地址。

Bash

```
# 在 test_client 内部执行
ping web_server
```

> **注意：** 默认的 `bridge` 网络（即不指定网络时使用的那个）是不支持容器名自动解析的，必须使用自定义网络。

------

#### 4. 管理与维护

如果你有已经运行的容器，想把它移动到新网络中，可以使用以下命令：

- **将容器连接到网络：**

  Bash

  ```
  docker network connect my_custom_network existing_container
  ```

- **查看网络详情（查看已连接的容器和 IP 分配）：**

  Bash

  ```
  docker network inspect my_custom_network
  ```

- **断开网络连接：**

  Bash

  ```
  docker network disconnect my_custom_network existing_container
  ```

------

#### 5. 在 Docker Compose 中使用 (推荐)

如果你使用 Compose，配置自定义网络会更加直观：

YAML

```
version: '3.8'
services:
  web:
    image: nginx
    networks:
      - backend_net

  db:
    image: mysql
    networks:
      - backend_net

networks:
  backend_net:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.20.0/24
```

------

#### 总结建议

- **隔离性**：将不同的业务模块（如前端、后端、数据库）放在不同的自定义网络中。
- **安全性**：自定义网络可以控制哪些容器可以互相通信，甚至可以通过配置 `iptables: false` 进一步限制（虽然这属于高级配置）。

# 七、Docker多阶段构建

Docker 的**多阶段构建（Multi-stage builds）**是优化 Dockerfile 的最强大工具之一。它的核心目的是：**在保持镜像尽可能小的同时，简化构建流程。**

在多阶段构建出现之前，我们通常需要准备两个文件：一个用于编译代码（包含所有依赖库和编译器），另一个用于运行程序（仅包含运行环境）。现在，你可以在一个 Dockerfile 中完成这一切。

------

### 1. 为什么需要多阶段构建？

如果你不使用多阶段构建，你的镜像往往会非常臃肿。

- **编译环境冗余：** 比如 Go 语言需要编译器，Java 需要 Maven/JDK，Node.js 需要 npm。一旦程序编译完成，这些工具就不再需要了。
- **安全性：** 镜像越小，包含的漏洞组件就越少，攻击面也随之减小。
- **层级结构：** 传统的构建方式会将所有中间层保留在最终镜像中。

------

### 2. 工作原理

多阶段构建允许你在 Dockerfile 中使用多个 `FROM` 指令。每个 `FROM` 指令都可以使用不同的基础镜像，并且标志着一个新阶段的开始。

你可以选择性地将**构建产物**（如可执行文件、静态资源）从一个阶段复制到另一个阶段，而舍弃掉不需要的中间过程。

------

### 3. 代码示例 (以 Go 语言为例)

这是一个典型的多阶段构建 Dockerfile：

Dockerfile

```
# --- 第一阶段：构建阶段 (Build Stage) ---
FROM golang:1.21-alpine AS builder

# 设置工作目录
WORKDIR /app

# 复制依赖文件并下载
COPY go.mod go.sum ./
RUN go mod download

# 复制源代码并编译
COPY . .
RUN go build -o myapp main.go

# --- 第二阶段：运行阶段 (Final Stage) ---
FROM alpine:latest

# 安装运行所需的最小依赖（可选）
RUN apk --no-cache add ca-certificates

WORKDIR /root/

# 【关键点】从 builder 阶段复制编译好的二进制文件
COPY --from=builder /app/myapp .

# 启动程序
CMD ["./myapp"]
```

#### 关键指令解析：

- **`AS builder`**：给这个构建阶段起个名字，方便后面引用。
- **`COPY --from=builder`**：这是精髓所在。它告诉 Docker 从名为 `builder` 的阶段中提取文件，而不是从宿主机复制。

------

### 4. 主要优势总结

| **优势**       | **说明**                                                     |
| -------------- | ------------------------------------------------------------ |
| **减小体积**   | 最终镜像只包含运行代码所需的最小环境（如从 500MB 减小到 20MB）。 |
| **维护简单**   | 所有的构建逻辑都在一个 Dockerfile 中，无需维护额外的构建脚本。 |
| **层级优化**   | 只保留最后一阶段的指令层，减少了镜像层数。                   |
| **CI/CD 友好** | 在流水线中只需要运行一个 `docker build` 命令即可。           |

------

### 5. 进阶技巧：只构建特定阶段

如果你想在调试时只运行某个阶段，可以使用 `--target` 参数：

Bash

```
docker build --target builder -t myapp:dev .
```

这在开发环境中非常有用，比如你只想运行测试阶段而不生成最终的发布镜像。

# 八、什么是Distroless镜像

简单来说，**Distroless 镜像**是一种极简的容器镜像，它**只包含你的应用程序及其运行所需的最小依赖**，而不包含任何传统的 Linux 发行版组件。

它由 Google 推广（GoogleContainerTools/distroless），核心理念是“**除了运行程序所必须的，其他一律不要**”。

------

### 1. 它与普通镜像（如 Ubuntu, Alpine）有什么区别？

传统的镜像（即使是极小的 Alpine）本质上都是一个缩减版的操作系统，而 Distroless 走得更远。

| **特性**                    | **传统镜像 (如 Ubuntu/Debian)** | **极简镜像 (如 Alpine)** | **Distroless 镜像**           |
| --------------------------- | ------------------------------- | ------------------------ | ----------------------------- |
| **Shell (bash/sh)**         | 有                              | 有                       | **无**                        |
| **包管理器 (apt/apk)**      | 有                              | 有                       | **无**                        |
| **常用工具 (ls, curl, vi)** | 有                              | 有                       | **无**                        |
| **攻击面**                  | 较大                            | 较小                     | **极小**                      |
| **镜像体积**                | 大 (100MB+)                     | 小 (5MB+)                | **极小 (通常比 Alpine 还小)** |

------

### 2. 为什么要使用 Distroless？

- **安全性 (Security)：** 这是最大的优势。由于没有 Shell 和包管理器，即便黑客通过漏洞进入了容器，他也无法执行 `ls` 查看文件，无法用 `curl` 下载木马，也无法用 `apt` 安装攻击工具。这极大地降低了“原地攻击”（Living off the land）的风险。
- **扫描噪音低：** 安全扫描器（如 Trivy）在扫描传统镜像时，经常会报出数百个来自 OS 工具库的 CVE 漏洞。Distroless 剔除了这些工具，让安全报告只关注你的代码本身。
- **启动速度快：** 镜像层级少、体积小，在 K8s 集群中拉取和启动的速度更快。

------

### 3. 使用中的挑战（缺点）

- **调试困难：** 因为容器里连 `ls` 和 `sh` 都没有，你不能 `docker exec -it` 进去看日志或查文件。
  - *解决方案：* 使用 Kubernetes 的 **Ephemeral Containers** (临时容器) 进行调试，或者使用专门的 `debug` 标签镜像。
- **构建复杂：** 你不能在 Dockerfile 里写 `RUN apt-get install`。你必须使用 **多阶段构建 (Multi-stage builds)**：在第一阶段（编译环境）编译好程序，在第二阶段直接将二进制文件复制到 Distroless 基础镜像中。

------

### 4. 总结与建议

**Distroless 并不适合所有人，但非常适合生产环境。**

- **什么时候用：** 你的应用已经稳定，准备部署到公网或对安全性要求极高的生产环境。
- **什么时候不用：** 开发阶段或需要频繁进入容器排查问题的环境。

**一个典型的 Go 语言多阶段构建示例：**

Dockerfile

```
# 阶段 1: 编译环境
FROM golang:1.21 AS build
COPY . /app
RUN cd /app && go build -o myapp

# 阶段 2: 运行环境 (Distroless)
FROM gcr.io/distroless/static-debian12
COPY --from=build /app/myapp /
CMD ["/myapp"]
```

# 九、Docker单点故障及解决方案

在使用 Docker 的生产环境中，“单点故障”（Single Point of Failure, SPOF）是一个非常关键的架构挑战。如果你的容器只跑在一台服务器上，一旦这台机器宕机、磁盘损坏或网络中断，整个业务就会直接下线。

以下是对 Docker 单点问题的深度分析及其主流解决方案：

------

## 1. Docker 单点问题的核心表现

- **宿主机故障：** 物理机或云服务器宕机，导致其上运行的所有容器停止服务。
- **Docker Daemon 故障：** Docker 守护进程崩溃或升级重启时，可能导致容器管理中断（虽然有 `live-restore` 可缓解，但无法处理严重崩溃）。
- **容器实例故障：** 单个容器因为 OOM（内存溢出）或程序 Bug 退出，没有自动拉起机制。
- **存储单点：** 数据卷（Volume）挂载在本地磁盘，机器坏了，数据也拿不出来。

------

## 2. 解决方案一：容器编排与集群化（最主流）

这是解决单点问题最彻底的方法，通过将多台服务器组成一个“资源池”，实现故障自动转移。

### Docker Swarm (原生支持)

Docker 自带的轻量级集群工具。

- **工作原理：** 多个节点组成集群，分为 Manager 和 Worker。
- **如何解决单点：** 当某个节点掉线，Manager 会自动在其他健康节点上启动新的容器副本。
- **特点：** 配置简单，适合中小规模应用。

### Kubernetes (K8s - 行业标准)

目前最强大的容器编排平台。

- **自愈能力：** 实时监控 Pod 状态，发现故障自动重启或飘移。
- **负载均衡：** 内置 Service 机制，自动分发流量，避免请求打到故障节点。
- **高可用架构：** 建议至少 3 个 Master 节点，确保“大脑”本身也不是单点。

------

## 3. 解决方案二：高可用负载均衡 (HA + Load Balancing)

即使后端有多个容器，如果流量入口只有一个，那入口就是单点。

- **Keepalived + LVS/Nginx/HAProxy：** 使用虚拟 IP (VIP)。两台负载均衡器通过 Keepalived 互相监控，如果主节点挂了，VIP 会自动飘移到备用节点。
- **云服务商负载均衡 (SLB/ELB)：** 利用云厂商提供的高可用负载均衡服务，通常它们在底层已经实现了跨可用区冗余。

------

## 4. 解决方案三：分布式存储

解决“数据单点”问题。如果容器飘移到了机器 B，但数据还在机器 A 的硬盘上，容器依然跑不起来。

- **网络文件系统：** 如 **NFS**（简单但不建议用于高并发）或 **Ceph**、**GlusterFS**（高性能分布式存储）。
- **云盘挂载：** 在云环境下，使用可以跨机器挂载的云存储卷。
- **数据库同步：** 尽量不在容器本地存重要数据，而是使用高可用的数据库集群（如 Redis Sentinel, MySQL MHA）。

------

## 5. 解决方案四：Docker 自身的容错配置

在非集群环境下，可以通过一些简单的配置降低风险：

- Restart Policy（重启策略）：

  启动容器时增加 --restart=always 或 --restart=unless-stopped。当容器挂掉或 Docker 重启时，它会自动尝试拉起容器。

- Live Restore：

  在 /etc/docker/daemon.json 中配置 "live-restore": true。这样当你重启 Docker 引擎进行维护时，运行中的容器不会停止。

------

## 总结与建议

| **场景**           | **推荐方案**                                          |
| ------------------ | ----------------------------------------------------- |
| **开发/测试环境**  | 设置 Docker `restart: always` 即可。                  |
| **中小型生产环境** | 使用 **Docker Swarm** + 外部 **NFS** 存储。           |
| **大型/复杂业务**  | 部署 **Kubernetes (K8s)** + 分布式存储 + 云负载均衡。 |

# 十、AWS解决Docker单点故障

在 Docker 的单机部署中，最核心的单点故障（SPOF）在于：**如果宿主机挂了，或者 Docker 守护进程崩溃，所有的服务都会中断。**

借助 AWS，你可以通过**横向扩展**和**托管服务**来彻底消除这些风险。以下是几种主流的架构方案，按复杂度由浅入深排列：

------

### 1. 进阶方案：使用 Amazon ECS + AWS Fargate（推荐）

这是解决单点问题最快、最有效的方式。它将容器调度和基础设施管理全权交给 AWS。

- **消除宿主机单点：** Fargate 是 Serverless（无服务器）容器环境。你不再需要管理 EC2 实例，AWS 会自动在多个可用区（Availability Zones）分发你的容器副本。
- **自愈能力：** ECS 服务（Service）会自动监控容器状态。如果一个容器挂了，ECS 会立即在健康的底层资源上拉起一个新的容器。
- **架构关键：**
  - **ALB (Application Load Balancer):** 位于最前端，负责流量分发。
  - **多可用区部署：** 将任务（Task）配置在至少两个子网中，确保即使一个数据中心断电，业务依然在线。

------

### 2. 标准方案：EC2 Auto Scaling + 负载均衡

如果你希望保留对宿主机的控制权，可以使用传统的集群模式。

- **Auto Scaling Group (ASG):** 设定最小运行实例数为 2。当某台机器宕机时，ASG 会根据健康检查结果自动补齐机器。
- **ELB 负载均衡：** 流量不直接指向容器 IP，而是指向 ELB。ELB 只会将请求转发给健康的容器。
- **消除单点：** 即使一台 EC2 彻底损坏，流量会自动切到另一台。

------

### 3. 企业级方案：Amazon EKS (Managed Kubernetes)

对于复杂的微服务架构，EKS 提供了更强大的容错机制。

- **控制面高可用：** AWS 自动管理 Kubernetes 的 Master 节点，并跨 3 个可用区进行冗余，消除了管理平台的单点。
- **Pod 级调度：** 利用 `podAntiAffinity`（亲和性配置），可以强制要求相同的服务不要跑在同一台物理机器上。
- **Karpenter / Cluster Autoscaler：** 当资源不足或节点异常时，自动扩容新的节点。

------

### 4. 数据层的单点消除

光解决容器运行是不够的，如果 Docker 挂载的是本地磁盘，机器坏了数据就丢了。

- **持久化存储：** 使用 **Amazon EFS (Elastic File System)**。它支持多实例同时挂载，且数据跨可用区冗余。容器漂移到新机器后，依然能访问相同的数据。
- **数据库：** 将数据库从 Docker 中移出，改用 **Amazon RDS**。开启 Multi-AZ（多可用区）部署，实现秒级自动切换。

------

### 方案对比表

| **维度**     | **Docker 单机**       | **ECS + Fargate**  | **EKS (Kubernetes)** |
| ------------ | --------------------- | ------------------ | -------------------- |
| **单点风险** | 极高 (机房/系统/进程) | 极低 (由 AWS 管理) | 极低 (控制面托管)    |
| **运维难度** | 低 (手动)             | 极低 (自动化)      | 高 (需学习 K8s)      |
| **自愈能力** | 无 (需手动重启)       | 强 (自动扩缩容)    | 极强 (声明式自愈)    |
| **适用场景** | 测试/开发             | **中小型生产环境** | 大型复杂微服务       |

### 建议操作步骤

1. **将镜像推送到 Amazon ECR：** 确保镜像有可靠的存储仓。
2. **创建 ECS 集群：** 选择 Fargate 模式以省去维护服务器的麻烦。
3. **配置 ALB：** 创建一个 Application Load Balancer，并设置健康检查路径。
4. **定义服务副本：** 在 ECS Service 中设置 `Desired Count` $\ge 2$，并勾选跨可用区部署。

# 十一、Docker Daemon Root 用户问题解决

在 Docker 的默认安装中，Docker Daemon（守护进程）始终以 `root` 用户身份运行。这意味着普通用户必须使用 `sudo` 才能执行 Docker 命令，这不仅不方便，还存在一定的安全风险。

针对这个问题，通常有两种主要的解决方法：**将用户添加到 Docker 用户组**（常用且方便）以及 **使用 Rootless 模式**（安全性最高）。

------

## 方案一：将非 Root 用户添加到 Docker 组

这是最常见的做法。Docker 安装后会创建一个名为 `docker` 的用户组，拥有该组权限的用户可以免 `sudo` 操作 Docker。

### 操作步骤：

1. **创建 Docker 组**（通常安装时已自动创建）：

   Bash

   ```
   sudo groupadd docker
   ```

2. **将当前用户加入 Docker 组**：

   Bash

   ```
   sudo usermod -aG docker $USER
   ```

3. 激活组更改：

   你可以注销并重新登录，或者运行以下命令立即生效：

   Bash

   ```
   newgrp docker
   ```

4. 验证：

   运行 docker run hello-world，如果不报错即表示成功。

> [!CAUTION]
>
> 安全警告： docker 组权限等同于 root 权限。如果用户可以访问 Docker 容器，他们可以通过挂载宿主机目录等方式获取宿主机的完全控制权。在生产环境中请谨慎授权。

------

## 方案二：使用 Rootless 模式（推荐用于高安全性需求）

如果你希望完全不以 `root` 身份运行 Docker Daemon，可以使用 Docker 官方提供的 **Rootless 模式**。这会将 Docker 运行在用户的命名空间内。

### 操作步骤：

1. 安装依赖：

   需要安装 newuidmap 和 newgidmap 工具。在 Ubuntu 上：

   Bash

   ```
   sudo apt-get install -y dbus-user-session uidmap
   ```

2. **运行安装脚本**：

   Bash

   ```
   curl -fsSL https://get.docker.com/rootless | sh
   ```

3. 配置环境变量：

   根据安装后的提示，在你的 ~/.bashrc 或 ~/.zshrc 中添加：

   Bash

   ```
   export PATH=/home/$USER/bin:$PATH
   export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock
   ```

4. **启动服务**：

   Bash

   ```
   systemctl --user start docker
   ```

**优点：** 即使容器被攻破，攻击者也无法获得宿主机的 `root` 权限。

------

## 方案三：通过 sudoers 配置精细化权限

如果你不想让用户拥有完整的 Docker 权限，可以通过修改 `/etc/sudoers` 文件，允许特定用户仅在执行 docker 命令时免密。

编辑 sudoers 文件（建议使用 `sudo visudo`）：

Plaintext

```
username ALL=(ALL) NOPASSWD: /usr/bin/docker
```

这样该用户执行 `sudo docker ...` 时将不再需要输入密码。

------

## 总结与对比

| **特性**       | **Docker 用户组**            | **Rootless 模式**              |
| -------------- | ---------------------------- | ------------------------------ |
| **操作便利性** | 极高，直接使用 `docker` 命令 | 一般，需额外配置环境变量       |
| **安全性**     | 较低（等同于 root）          | 极高（隔离 root 权限）         |
| **功能限制**   | 无限制                       | 无法使用某些网络功能或特权模式 |
| **适用场景**   | 开发环境、受信任的多用户机   | 生产环境、对安全极其敏感的系统 |

# 十二、AWS 中 Docker 资源限制配置

在 AWS 中限制 Docker 容器的资源（CPU 和内存），主要取决于你使用的服务（如 ECS, Fargate, EC2 或 Lambda）。

最常见的场景是在 **Amazon ECS (Elastic Container Service)** 中通过 **任务定义 (Task Definition)** 进行配置。

------

## 1. 在 Amazon ECS 中配置 (Fargate & EC2)

在 ECS 中，你可以在两个层面设置限制：**任务级别 (Task level)** 和 **容器级别 (Container level)**。

### 内存限制 (Memory Limits)

ECS 提供了两种内存限制方式，对应 Docker 的参数：

- **硬限制 (Hard Limit / `memory`)**：
  - **对应参数**：`memory`
  - **行为**：如果容器尝试超过此内存值，它会被 Docker 守护进程直接 **Kill (OOM Killer)**。
- **软限制 (Soft Limit / `memoryReservation`)**：
  - **对应参数**：`memoryReservation`
  - **行为**：这是容器预留的最小内存。当主机内存充足时，容器可以超过此限制；但当主机资源紧张时，系统会尝试将容器限制回此值。

### CPU 限制 (CPU Units)

AWS 使用 **CPU Units** 作为单位（$1 vCPU = 1024 CPU units$）。

- **Fargate**：必须在任务级别指定 CPU，容器共享该总量。
- **EC2**：可以在容器级别指定 `cpu` 权重，或者指定 `cpu` 硬限制。

------

## 2. 具体的配置方法

### 方法 A：通过 AWS 管理控制台 (Console)

1. 打开 **ECS 控制台**，选择 **Task Definitions**。
2. 创建新修订版本或新定义。
3. **任务级别限制**：在“Task size”部分，选择该任务的总 CPU 和内存（如果是 Fargate，这是必填项）。
4. **容器级别限制**：点击具体的容器定义，在 **Environment** 或 **Resource limits** 栏目下：
   - 输入 **Soft limit** (内存预留)。
   - 输入 **Hard limit** (内存上限)。
   - 输入 **CPU units**。

### 方法 B：通过 JSON 任务定义文件

如果你使用基础设施即代码（Terraform, CloudFormation 或 CLI），配置如下：

JSON

```
{
  "containerDefinitions": [
    {
      "name": "my-app",
      "image": "my-docker-image",
      "cpu": 256,         // 容器分配的 CPU units (0.25 vCPU)
      "memory": 512,      // 硬限制 (MiB)
      "memoryReservation": 256, // 软限制 (MiB)
      "essential": true
    }
  ],
  "cpu": "512",           // 任务级别总 CPU
  "memory": "1024"        // 任务级别总内存
}
```

------

## 3. 不同场景的选择建议

| **场景**              | **配置建议**                                                 |
| --------------------- | ------------------------------------------------------------ |
| **AWS Fargate**       | **强制**要求设置任务级 CPU 和内存。建议容器级不设硬限制，让应用共享任务资源，防止单个容器因瞬时峰值被杀。 |
| **Amazon EC2 (ECS)**  | 建议设置 **Soft Limit** 以保证基础运行，配合 **Hard Limit** 防止某个容器因内存泄漏拖垮整台 EC2 实例。 |
| **AWS Lambda (容器)** | 资源由 Lambda 函数的“Memory”设置决定。你分配的内存越多，按比例获得的 CPU 性能也越高（Lambda 不支持单独设置容器内部的 Docker 限制）。 |

------

### 最佳实践提示

- **监控**：使用 **CloudWatch Container Insights** 查看容器的实际 `MemoryUtilized` 和 `CpuUtilized`。
- **OOM 排查**：如果容器频繁重启并显示 `OutOfMemoryError`，请检查是否触发了 **Hard Limit**。
- **分级预留**：通常设置 `memoryReservation` 为正常运行值，`memory`（硬限制）为正常值的 1.5 - 2 倍。

# 十三、EC2 运行 Docker 的正确姿势

这是一个非常经典的问题。简单直接的回答是：**在 AWS EC2 上运行 Docker 不仅不是错误的，反而目前工业界最主流、最成熟的实践方式之一。**

虽然从表面上看，在虚拟机（VM）里运行容器（Container）似乎多了一层“虚拟化”，但这种架构背后的逻辑其实非常扎实。

------

### 1. 为什么“虚拟机 + 容器”是黄金搭档？

虽然 Docker 可以在裸机（Bare Metal）上运行，但在云环境下，将它跑在 EC2 上的原因主要有以下几点：

- **强隔离性与安全性：** Docker 容器共享宿主机的内核。如果在裸机上内核出现漏洞，容器间可能发生“逃逸”。EC2（虚拟机）通过硬件虚拟化提供了一层坚固的隔离边界，即使一个 VM 里的容器出问题，也不会威胁到其他用户的 VM。
- **资源管理的灵活性：** EC2 让你能快速调整底层资源。如果你的 Docker 应用需要更多内存，你可以轻松从 `t3.medium` 升级到 `r5.large`，而不需要搬动物理服务器。
- **成熟的生态支撑：** AWS 的很多服务（如 EBS 存储、VPC 网络、IAM 身份权限）都是绑定在 EC2 实例上的。在 EC2 上跑 Docker，可以无缝利用这些云原生功能。

### 2. 性能损耗真的很大吗？

这是一个常见的误区。

- **计算损耗极小：** 现代 CPU 都有硬件加速指令（如 Intel VT-x），虚拟机处理指令的损耗通常在 **5% 以内**。
- **容器几乎“零”损耗：** Docker 并不是虚拟机，它只是宿主机上的一个进程。它直接运行在 EC2 的 Linux 内核上，不会再额外消耗一次虚拟化性能。

### 3. 何时该考虑其他方案？

虽然在 EC2 上手动装 Docker 没错，但随着规模变大，你会发现“管理 EC2 本身”变成了负担（比如需要自己升级操作系统补丁、扩容等）。这时候你可以考虑：

| **方案**          | **特点**                                                     | **适用场景**                           |
| ----------------- | ------------------------------------------------------------ | -------------------------------------- |
| **EC2 + Docker**  | 完全控制权，想怎么配置都行。                                 | 实验环境、小型应用、需要特定内核配置。 |
| **AWS ECS / EKS** | 容器编排服务（底层仍可基于 EC2）。                           | 大规模集群管理、微服务架构。           |
| **AWS Fargate**   | **无服务器（Serverless）容器**。你不再能看到 EC2，只需给容器分配 CPU 和内存。 | 追求运维极致简单，不想管服务器。       |

### 4. 总结

在虚拟机上跑容器是**“逻辑隔离（Docker）”与“物理隔离（VM）”的结合**。这种组合提供了目前最平衡的安全性、可移植性和性能。

**一句话建议：** 如果你只是刚开始或者需要深度定制系统，用 EC2 跑 Docker 没问题；如果你希望以后不用管服务器补丁和维护，可以看看 **AWS Fargate**。

# 十四、Docker 容器性能监控指南

Docker 容器性能监控是一个多维度的任务，通常根据监控的深度（实时 vs 历史）和规模（单机 vs 集群）分为不同的方案。

以下是针对 Docker 容器性能监控的全面指南：

------

## 1. 核心监控指标 (Metrics)

在监控 Docker 时，以下四类指标最为关键：

- **CPU 指标**：利用率（Usage）、配额限制（Quota）、节流时间（Throttling）。
- **内存指标**：已用内存（Usage）、缓存（Cache）、限制（Limit）、SWAP 使用情况。
- **网络 I/O**：入站/出站流量（RX/TX）、丢包数、网络错误。
- **磁盘 I/O**：读写字节数（Read/Write Bytes）、操作次数（IOPS）。

------

## 2. 常用监控方案对比

根据你的需求，可以选择以下三种主流方案：

### 方案 A：入门级 - Docker 内置命令 (`docker stats`)

适用于单机环境下的**快速查看**和**实时诊断**。

- **用法**：在终端输入 `docker stats`。
- **优点**：无需安装任何额外软件，原生支持。
- **缺点**：无法保存历史数据，没有可视化图表，不支持告警。

### 方案 B：主流开源套件 - cAdvisor + Prometheus + Grafana

这是目前生产环境中最流行的**标准化开源方案**。

1. **cAdvisor (Google开源)**：负责从宿主机采集容器的实时资源消耗数据。
2. **Prometheus**：负责定期抓取 cAdvisor 的数据并进行历史存储。
3. **Grafana**：负责将 Prometheus 中的数据转化为精美的仪表盘。

### 方案 C：全栈/商业方案 (SaaS)

适用于需要高可用、自动化告警和深度链路追踪的企业。

- **Datadog / New Relic**：功能强大，支持自动发现，但成本较高。
- **Zabbix**：传统监控工具，通过 Docker 插件也能实现较好的监控效果。

------

## 3. 快速上手：使用 cAdvisor 进行图形化监控

如果你想在几分钟内看到可视化界面，可以直接运行 Google 的 **cAdvisor** 容器：

Bash

```
docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --publish=8080:8080 \
  --detach=true \
  --name=cadvisor \
  google/cadvisor:latest
```

运行后，访问 `http://localhost:8080` 即可看到当前宿主机上所有容器的实时性能图表。

------

## 4. 总结与建议

| **场景**            | **推荐方案**                             |
| ------------------- | ---------------------------------------- |
| **临时查看**        | 使用 `docker stats` 命令行               |
| **单机可视化**      | 部署 `cAdvisor` 查看 Web 界面            |
| **中大规模生产**    | 搭建 `Prometheus + Grafana + cAdvisor`   |
| **Kubernetes 集群** | 使用 `Kube-state-metrics` + `Prometheus` |
