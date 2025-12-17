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
   git clone https://github.com/iam-veeramalla/Docker-Zero-to-Hero
   cd  Docker-Zero-to-Hero/examples/first-docker-file
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
   docker build -t bob0516/my-first-docker-image:latest .
   ```

   验证容器被正确生成：

   ```
   docker images
   ```

4. 运行容器

   ```
   docker run -it bob0516/my-first-docker-image
   ```

5. 推送image到注册中心

   登录注册中心，本例中是hub.docker.com这个公用注册中心。

   ```
   docker login
   ```

   推送image：

   ```
   docker push bob0516/my-first-docker-image
   ```

   可以看到image已经推送成功：

   https://hub.docker.com/r/bob0516/my-first-docker-image

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
   FROM eclipse-temurin:8-jre-alpine
   
   # 设置工作目录
   WORKDIR /app
   
   # 从构建阶段拷贝生成的 jar 包到当前镜像
   # 请根据你 pom.xml 中的 artifactId 和 version 修改 jar 包名称
   COPY --from=build /app/target/*.jar docker-java-web-app-0.0.1-SNAPSHOT.jar
   
   # 暴露项目端口（假设为 8080）
   # EXPOSE 并不实际开放端口，它更像是一种文档说明，告诉开发者或系统该应用预期监听哪个端口
   EXPOSE 8080
   
   # 启动命令
   ENTRYPOINT ["java", "-jar", "docker-java-web-app-0.0.1-SNAPSHOT.jar"]
   ```

   这个Dockfile既是一个Java应用，也是一个多阶段构建的例子

3. Build

   ```
   docker build -t bob0516/docker-java-web-app:latest .
   ```

4. Run

   ```
   docker run -d -p 8080:8080 bob0516/docker-java-web-app:latest
   ```

5. Visit by browserDockfile

   访问http://公网IP:8080，站点正常显示。

# FAQ
