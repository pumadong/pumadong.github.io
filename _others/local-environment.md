---
layout: post
title: 本机环境
date: 2025-12-12 18:30:00 +0800  # 标准格式
description: "这是我专门为这篇文章写的精简摘要，直接显示在列表页。"
excerpt: "这接显示在列表页。"
---

# VMware Fusion虚拟机
1. 桥接模式：192.168.1.200

2. 宿主机：Mac 12.7，192.168.1.100

3. Redis：192.168.1.200:6379

4. MySQL：192.168.1.200:3309  root/123456

   

# Docker Desktop

简单来说，**Docker Desktop 的最新版本（v4.55.0+）官方已经不再支持 macOS 12 (Monterey) 了。**

Docker 的官方政策是支持 **最新的三个主要 macOS 版本**。由于目前（2025年底）主流版本为 macOS 15 (Tahoe/Sequoia 相关版本)、14 (Sonoma) 和 13 (Ventura)，因此 macOS 12 已被移出官方支持范围。

以下是详细的兼容性说明和建议：

------

## 1. 为什么最新版不支持？

根据 Docker 官方文档的更新记录：

- **从版本 4.49.0 开始**：Docker Desktop 的安装和更新要求最低版本提升至 **macOS 14 (Sonoma)**。

- **在 4.48.0 版本中**：官方明确指出对 macOS 13 的支持已结束，下一版本将强制要求 macOS 14。

- **macOS 12 (Monterey)**：支持更早之前就已经停止。

  

## 2. macOS 12.7 还能用哪个版本？

虽然最新版无法安装，但你可以通过安装 **旧版本** 来继续使用。根据社区反馈和发布记录，以下版本通常是 macOS 12 用户的较稳选择：

- **推荐版本：Docker Desktop 4.38.0 或更早版本。**

- 部分用户反馈 **4.42.0** 可能是最后几个能在 Monterey 上尝试运行的版本，但可能会遇到不稳定的情况。

  

## 3. 如何解决安装问题？

如果你必须在 macOS 12.7 上运行 Docker，请参考以下操作：

1. **禁止自动更新**：安装旧版本后，务必在设置中关闭“Check for updates”，否则系统会自动下载不兼容的新版导致崩溃。

2. **手动下载旧版**：你可以前往 [Docker Desktop Release Notes](https://docs.docker.com/desktop/release-notes/) 页面，向下滚动寻找 2024 年上半年或更早的发布版本下载 DMG 安装包

   - **Intel 处理器:** [Docker Desktop 4.25.0 (Intel)](https://desktop.docker.com/mac/main/amd64/126437/Docker.dmg)
   - **备选版本：** 如果 4.25.0 在您的系统上运行异常，可以尝试更保守的 **v4.24.2**（这是最后一个明确对旧系统有极佳兼容性的版本）：
     - [4.24.2 for Intel](https://desktop.docker.com/mac/main/amd64/124339/Docker.dmg)
   
3. **替代方案**：如果旧版 Docker Desktop 运行缓慢，可以考虑轻量级的替代品，如 **OrbStack** 或 **Colima**，它们往往对旧版 macOS 有更好的兼容性。

4. **配置镜像加速**

   **配置步骤**

   1. **打开设置**：点击顶部菜单栏的 Docker 图标，选择 **Settings** (设置)。
   
   2. **定位配置页**：在左侧导航栏中选择 **Docker Engine**。
   
   3. **修改 JSON**：在右侧的 JSON 编辑框中，找到 `registry-mirrors` 键（如果没有则手动添加）。将镜像地址填入数组中。
   
   4. **保存并重启**：点击右下角的 **Apply & Restart**。
   
      
   
   **配置示例：**
   
   JSON
   
   ```
   {
     "registry-mirrors": [
       "https://docker.1ms.run",
       "https://docker.xuanyuan.me",
       "https://docker.m.daocloud.io"
     ],
     "builder": {
       "gc": {
         "defaultKeepStorage": "20GB",
         "enabled": true
       }
     },
     "experimental": false
   }
   ```
   
   ------
   
   **2025 推荐可用加速器地址**
   
   目前国内直接访问 Docker Hub 极不稳定，以下是近期收集的可用源：
   
   | **来源**       | **镜像地址**                           | **备注**                      |
   | -------------- | -------------------------------------- | ----------------------------- |
   | **1ms (推荐)** | `https://docker.1ms.run`               | 2025 活跃，速度较快           |
   | **轩辕镜像**   | `https://docker.xuanyuan.me`           | 个人维护，稳定性好            |
   | **DaoCloud**   | `https://docker.m.daocloud.io`         | 老牌镜像站，可作备选          |
   | **阿里云**     | `https://<你的ID>.mirror.aliyuncs.com` | 需登录阿里云控制台获取专属 ID |

# Idea

## Spring Initializr ServerURL

构建项目时，Spring Initializr Server URL，由默认的start.spring.io，更换为start.aliyun.com是更好的选择。
[Spring Initializr 构建SpringBoot项目时Server URL选择start.spring.io和start.aliyun.com的区别 原创](https://blog.csdn.net/dengxin686868/article/details/137127524)

## Lombok和Java版本兼容问题

java: java.lang.NoSuchFieldError: Class com.sun.tools.javac.tree.JCTree$JCImport does not have member field 'com.sun.tools.javac.tree.JCTree qualid'

这是lombok和java版本的冲突问题，参考这里：[JDK与Lombok版本兼容性冲突解决方案及实践](https://comate.baidu.com/zh/page/9x1taz7ac6m)。

通过homebrew安装JDK11解决：[LINK](https://www.google.com/search?q=homebrew+install+java+11&sca_esv=5cb444661cf9bcf8&sxsrf=AE3TifMHnSfqVuYYyEFZHKOc76D5hfDEAQ%3A1764937444646&source=hp&ei=5M4yadG9JdbdkPIP_fm6gQk&iflsig=AOw8s4IAAAAAaTLc9LqY2PgOYx0S4FYCjdl_s3dsha3N&oq=homebrew+&gs_lp=Egdnd3Mtd2l6Iglob21lYnJldyAqAggIMgsQABiABBiRAhiKBTILEAAYgAQYkQIYigUyCxAAGIAEGJECGIoFMgoQABiABBhDGIoFMg0QABiABBixAxhDGIoFMgsQABiABBiRAhiKBTIKEAAYgAQYQxiKBTIIEAAYgAQYsQMyChAAGIAEGEMYigUyChAAGIAEGEMYigVI9klQAFj2EHABeACQAQCYAeoCoAGgFqoBBTItNS41uAEDyAEA-AEBmAILoALYFsICChAjGPAFGCcYngbCAgoQIxiABBgnGIoFwgIEECMYJ8ICERAuGIAEGLEDGNEDGIMBGMcBwgIOEAAYgAQYsQMYgwEYigXCAg4QLhiABBixAxiDARiKBcICDRAAGIAEGJECGIoFGArCAgsQABiABBixAxiDAcICBRAAGIAEwgIOEC4YgAQYsQMY0QMYxwHCAg0QLhiABBhDGOUEGIoFwgIQEAAYgAQYsQMYQxiDARiKBcICCxAuGIAEGMcBGK8BmAMAkgcHMS4wLjUuNaAH8FqyBwUyLTUuNbgH1BbCBwcwLjMuNy4xyAco&sclient=gws-wiz)。

- brew install openjdk@11
- /usr/local/Cellar/openjdk@11/11.0.29/libexec/openjdk.jdk/Contents/Home
- /usr/local/Cellar/openjdk/23.0.2/libexec/openjdk.jdk/Contents/Home
- 在Idea中新增JDK时，通过command+shift+G输入以上路径，更换JDK
- 在本地通过更改~/.bash_profile更换java版本：export PATH="/usr/local/opt/openjdk@11/bin:$PATH"
