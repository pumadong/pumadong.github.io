---

layout: single
title: 读一本书《Netty in Action V5》-Chapter 2
permalink: /mw/netty-in-action-2.html

classes: wide

author: Bob Dong

---

# 前言

本篇基于MEAP Edition进行翻译，MEAP即Manning Early Access Program，[Manning](https://www.manning.com/)是一个出版社的名字，EAP是早期预览版。

我计划翻译这本书的时候，已经有了Netty in Action V10，但是我感觉从基础的Java NIO 讲起，能有一些对比，更容易理解，所以就翻译了V5。另外早期预览版，和正式出版相比，代码和文字的错误，都较多一些，要注意一下。

Norman Maurer，是英文原著的作者，Netty 核心开发者之一，目前就职于苹果公司，Norman Maurer的个人主页：http://normanmaurer.me/。

本书的调试过之后，可运行的源码，都可以Github下载：https://github.com/pumadong/netty-in-action-v5 。

在本书的翻译过程中，感觉写的有些啰嗦，在代码和文字中都有一些错误，力求在翻译的过程中根据自己的学习和理解，修复这些代码和文字错误。



**读本书之前，先让我们简单了解，Netty是什么？**

Netty是一套框架，万紫千红的Java生态中的一套通讯框架，一套传输层的通讯框架。

这套框架，比较偏底层一些（传输层，你就说Socket层也行），所以比较少被应用开发直接拿来使用，而是在一些广为流传的RPC服务治理框架中，作为底层的通讯框架。

做通讯开发的工程师少，做应用开发，特别是数据库操作的工程师众多；所以，做个类比，用JDK中的SQL包可以做数据库开发，用MyBatis框架也可以做数据库开发，用JDK中的BIO/NIO/NIO.2可以做通讯开发，用Netty也可以做通讯开发；通讯开发中的Netty框架，就像数据库开发中的MyBatis框架。



## 第2章 Your First Netty Application



**本章包含内容**



- 获取Netty的最新版本
- 安装开发环境运行例程
- 创建Netty客户端和服务端
- 拦截和处理错误
- 编译和运行Netty客户端和服务端



本章，通过对Netty核心内容的入门介绍为本书的其他章节做准备。其中一项内容是学习怎么样利用Netty拦截和处理异常，当我们开始使用Netty需要调试问题的时候，这是非常重要的。本章也介绍了其他的核心内容，像客户端和服务端启动，通过[通道](https://so.csdn.net/so/search?q=通道&spm=1001.2101.3001.7020)处理器实现的解耦。为了将来的章节提供一个基础，你会通过Netty建立一个互相通讯的客户端和服务端。

当然，首先你要建立一个开发环境。



### 2.1 建立开发环境



通过下面的步骤，建立一个开发环境：



- \1. 安装 Java 用来编译和运行例程。
  你的操作系统可能包含JDK，如果已经包含，可以跳过这一步。如果你需要安装 Java ，可以通过 Http://java.com 获取最新版本。请确保安装了JDK（不是JRE），因为需要JDK来编译代码。如果你需要安装文档，请参考上面的 Java 站点。

- \2. 下载和安装Maven。

  Maven 是一个依赖管理工具，它使依赖跟踪更为容易。可以通过Maven站点 http://maven.apache.org 下载压缩包。本书出版时，最新的版本是 3.0.5。
  请确保下载了适合你操作系统的压缩包。对于Windows，是.zip压缩包，对于类Unix的操作系统（比如Linxu和Mac OSX)，下载.bz2压缩包。
  下载文件后，解压缩到一个文件夹，按照适合你操作系统的方式进行配置。



**注意：**因为Netty源码用Maven，和这类似，我们的例子代码也都使用Maven。

下面的列表演示了Mac操作系统中安装Maven时的文件列表。

![netty-in-action](images/java-netty-in-action-1.png)

解压压缩包后，可以把mvn命令加到PATH路径，以便更方便的执行Maven。否则的话，你就需要使用全路径来运行Maven。

再强调一下，怎么把mvn命令加到PATH路径，依赖于你使用的操作系统。

对于类Unix操作系统，一般是配置.profile或者.bashrc文件。

在Windows操作系统中，可以通过系统配置面板设置PATH路径。

Maven用它的标准目录布局组织工程。你可以在Maven挂网学习这个布局的更多知识。我仅会关注为了演示例程需要的重要目录。表单2.1演示了Maven工程需要的最主要的目录。

![netty-in-action](images/java-netty-in-action-2.png)

Maven使用配置文件pom.xml进行项目管理，像下面的列表出示的一样。pom.xml需要被放置在项目的根目录下。

```
<?xml version="1.0" encoding="ISO-8859-15"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
http://maven.apache.org/maven-v4_0_0.xsd">
 <modelVersion>4.0.0</modelVersion>
 <groupId>com.manning.nettyinaction</groupId>
 <artifactId>netty-in-action</artifactId>
 <name>netty-in-action</name>
 <version>0.1-SNAPSHOT</version>
 <build>
   <plugins>
     <plugin>
       <groupId>org.apache.maven.plugins</groupId>
       <artifactId>maven-compiler-plugin</artifactId>
       <version>2.5.1</version>
       <configuration>
         <optimize>true</optimize>
         <source>1.6</source>
         <target>1.6</target>
       </configuration>
     </plugin>
   </plugins>
 </build>
 <dependencies>
   <dependency>
     <groupId>io.netty</groupId>
     <artifactId>netty-all</artifactId>
     <version>4.0.0.Final</version>
   </dependency>
 </dependencies>
</project>
```

你需要建立一个目录结构，至少有一个pom.xml和一个src/main/java目录。在pom.xml中，每一个dependency标签都是一个类库的容器，将会被Maven自动包括。这个例子中仅仅列出了我们的例程需要的Netty类库，实际上你可以根据你的需要来配置更多dependency。





### 2.2 Netty客户端和服务端概述



本节的目标是通过构建一个完整的Netty客户端和服务端程序进行指导。典型来说，你可能仅仅对于开发服务端有兴趣，像供客户端浏览的HTTP服务器，然而，如果你学习了解了客户端和服务端实现的全部的细节，你会得到一个更清晰的视图。

下面这个图片示例了Netty应用，figure 2.2.

![netty-in-action](images/java-netty-in-action-3.png)

\#1 连接服务端的客户端

\#2 建立连接去发送/接收数据

\#3 处理所有客户端请求的服务端

从 figure 2.1 中我们可以看到，我们将要开发的Netty服务端，会自动处理几个并发的客户端。理论上来说，客户端数目仅仅受限于系统资源和JDK的限制。

为了更容易理解，对上面的图示想象一下，在一个河谷或者山上，大喊几句，然后听到回声。在这个场景中，你是客户端，大山是服务端。到达大山大喊，你建立了一个连接。大喊的动作类似于Netty客户端发送数据到服务端。听到你的声音并且返回回声，类似于Netty服务端收到并返回你发送的数据。离开大山，连接被中断，但是你可以返回重新连接到服务端并发送数据。

虽然返回同样的数据返回给客户端不是典型案例，但是这个在客户端和服务端发送接收数据是通用的。本章后面的例子将会用越来越复杂的例子验证这些。

下面的一些章节，将会演示通过Netty建立这个echo客户端和服务端的过程。



### 2.3 写一个Echo程序服务端



写一个Netty服务端包含两个主要的部分：



- 启动--配置服务端属性，像线程和端口
- 实现服务端处理器--建立包含商业逻辑的组件，它决定，当连接被建立，收到数据后，应该怎么处理





#### 2.3.1 启动服务端



你通过生成一个ServerBootstrap类的实例启动一个服务端。这个实例被配置可选项，像端口，线程模型，事件循环，商业逻辑处理器（在这儿例子中，它返回收到的数据，我们可以根据实际需要开发这个商业逻辑处理器）。

Listing 2.3 Main Class for the server

<https://github.com/pumadong/netty-in-action-v5/blob/master/src/main/java/chapter2/EchoServer.java>

```
package chapter2;
 
import java.net.InetSocketAddress;
 
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
 
/**
 * EchoServer
 */
public class EchoServer {
    private final int port;
 
    public EchoServer(int port) {
        this.port = port;
    }
 
    public void start() throws Exception {
        // #3 Create the EventLoopGroup
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            // #4 Create the ServerBootstrap
            ServerBootstrap b = new ServerBootstrap();
            b.group(group)
                // #5 Specify the use of an NIO transport Channel
                .channel(NioServerSocketChannel.class)  
                // #6 Set the socket address using the selected port
                .localAddress(new InetSocketAddress(port))  
                // #7 Add an EchoServerHandler to the Channel's ChannelPipeline
                .childHandler(new ChannelInitializer<SocketChannel>() { 
                    @Override
                    public void initChannel(SocketChannel ch) throws Exception {
                        ch.pipeline().addLast(new EchoServerHandler());
                    }
                });
            // #8 Bind the server; sync waits for the server to close
            ChannelFuture f = b.bind().sync();
            System.out.println(EchoServer.class.getName() + " started and listen on " + f.channel().localAddress());
            // #9 Close the channel and block until it is closed
            f.channel().closeFuture().sync();
            System.out.println(EchoServer.class.getName() + " close");
        } finally {
            // #10 Shutdown the EventLoopGroup, which releases all resources.
            group.shutdownGracefully().sync();
            System.out.println(EchoServer.class.getName() + " finally");
        }
    }
 
    public static void main(String[] args) throws Exception {
        
//         if (args.length != 1) {
//         System.err.println("Usage: " + EchoServer.class.getSimpleName() +
//         " <port>");
//         }
//         // #1 Set the port value (throws a NumberFormatException if the port
//         argument is malformed)
//         int port = Integer.parseInt(args[0]);
        
        int port = 8888;
        // #2 all the server's start() method.
        new EchoServer(port).start();
    }
}
```

这个例子中，代码生成了一个ServerBootstrap实例（4）。因为我们使用了NIO通道，所以我们定义了NioEventLoopGroup（3）去接受和处理新连接，定义了NioServerSocketChannel（5）作为通道类型。然后，我们通过InetSocketAddress设置当地的监听端口（6）。服务端将会绑定这个端口监听新的连接请求。

第七步是关键一步：这里我们使用了一个特别的类，ChannelInitializer。当一个新连接被接受后，一个新的子通道将会被生成，这个ChannelInitializer将会加一个EchoServerHandler的实例到通道的ChannelPipeline中。像我们早先解释的那样，当有消息到达时，这个处理器将会被通知。

当要求NIO是可扩展的，合适的配置不是那么容易的。一般来说，实现一个正确运行的多线程也不是非常容易的。幸运的是，Netty的设计封装了大部分的复杂性，特别是通过的一些通用抽象，像EventLoopGroup，SocketChannel 和 ChannelInitializer，这些中的每一个，都会在第三章中详细讨论。

在第八步中，我们绑定服务端，一直到绑定完成（sync方法的调用会让当前线程阻塞直到完成）。在第九步中，应用会阻塞，直到服务端通道关闭（因为我们在通道的CloseFuture上调用了sync方法）。在第十步中，try的finally中，我们可以关闭EventLoopGroup，释放所有的资源，包括所有生成的线程。

在这个例子中使用了NIO，因为它是当前最广泛使用的传输，因为它的可扩展性和彻底的异步。当然，使用一个不同的传输也是可以的。例如，如果这个例子使用OIO传输，我们将会定义 OioServerSocketChannel和OioEventLoopGroup。Netty的架构，包含关于传输的更多信息，第4章将会覆盖这些内容。接下来，让我们复习一下我们刚刚学习的服务端实现中的重要的步骤。



**我们再强调一遍重要的事情：**



- 生成一个ServerBootstrap实例去启动服务端，以及稍后进行绑定端口
- 生成并分配一个NioEventLoopGroup实例去处理事件过程，像接受新连接和读写数据
- 定义一个服务端要绑定的InetSocketAddress
- 用EchoServerHandler实例初始化每个新Channel
- 最后，调用ServerBootstrap.bind()去绑定服务端



#### 2.3.2 实现服务端商业逻辑



Netty使用了前面讨论的future和callback，允许你使用不同的事件类型。后面会进行详细讨论，现在，让我们关注收到的数据。为了处理数据，你的channel处理器必须扩展ChannelInboundHandlerAdapter，重写messageReceived方法。每次接收到信息后，这个方法都会被调用，在这个案例中收到的信息是字节类型。

下面的代码，演示了放置商业逻辑，本例中即发送收到的数据给客户端。

<https://github.com/pumadong/netty-in-action-v5/blob/master/src/main/java/chapter2/EchoServerHandler.java>

```
package chapter2;
 
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelFutureListener;
import io.netty.channel.ChannelHandler.Sharable;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.util.CharsetUtil;
 
/**
 * EchoServer业务逻辑处理程序
 */
// #1 Annotate with @Sharable to share between channels
@Sharable
public class EchoServerHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        System.out.println("Server received:" + ((ByteBuf)msg).toString(CharsetUtil.UTF_8));
        // #2 Write the received messages back . Be aware that this will not
        // flush the messages to the remote peer yet.
        ctx.write(msg);
    }
 
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        // #3 Flush all previous written messages (that are pending) to the
        // remote peer, and close the channel after the operation is complete.
        ctx.writeAndFlush(Unpooled.EMPTY_BUFFER).addListener(ChannelFutureListener.CLOSE);
    }
 
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        // #4 Log exception
        cause.printStackTrace();
        // #5 Close channel on exception
        ctx.close();
    }
}
```

Netty使用channel处理器做了一个伟大的关注分离，使增加、更新、移除商业逻辑更容易。这个处理器是直接的，它的每一个方法可以被重写，以"hook"进入数据循环的一部分，但是channelRead方法需要被重写。



#### 2.3.3 拦截异常



重写 channelRead 方法时，你可能注意到 exceptionCaught 方法也被重写了。这是用来处理异常的，包括任何 Throwable 的子类型。在这个案例中，我记录日志，并且关闭了客户端连接，虽然这个连接可能在一个未知状态。一般来说，这就是你能做的所有的事了，但是有的时候，从错误恢复几乎是不可能的，所以需要你建立一个更智能的实现。你应该注意，你必须有至少一个 ChannelHandler 实现这个方法，提供一个方式去处理所有的错误。

Netty 拦截异常的方式，让处理不同线程的错误更容易。单个线程不能捕获的异常，都通过同样简单的集中的 API 来处理。

有一些你应该注意的，其他的 ChannelHandler 子类型和实现，如果你计划开发一个真实世界的应用，或者用 Netty 写一个框架，稍后我会谈到这些。现在，记住 ChannelHandler 实现将会为不同类型的事件调用，你能实现或者扩展它们去进入事件循环。



### 2.4 写一个Echo程序客户端



现在，服务端代码写好了，我们写一个客户端来使用它。



- 客户端包括下面的任务：
- 连接服务端
- 发送数据
- 等待，直到从服务端接收到同样的数据
- 关闭连接



带着这些目的，我们写实际的逻辑，和上一节中实现服务端一样。



#### 2.4.1 启动客户端



如下面代码所示，启动客户端和启动服务端同样简单。客户端启动，需要一个主机和一个端口去连接，不像服务端启动，仅仅需要一个端口即可。

Listing 2.5 Main class for the client

```
package chapter2;
 
import java.net.InetSocketAddress;
 
import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
 
/**
 * EchoClient
 */
public class EchoClient {
    private final String host;
    private final int port;
 
    public EchoClient(String host, int port) {
        this.host = host;
        this.port = port;
    }
 
    public void start() throws Exception {
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            // #1 Create bootstrap for client
            Bootstrap b = new Bootstrap();
            // #2 Specify EventLoopGroup to handle client events. NioEventLoopGroup is used, as the NIO-Transport should be used
            b.group(group)
                // #3 Specify channel type; use correct one for NIO-Transport
                .channel(NioSocketChannel.class)
                // #4 Set InetSocketAddress to which client connects
                .remoteAddress(new InetSocketAddress(host, port))
                // #5 Specify ChannelHandler, using ChannelInitializer, called once connection established and channel created
                .handler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    public void initChannel(SocketChannel ch) throws Exception {
                        // #6 Add EchoClientHandler to ChannelPipeline that belongs to channel. 
                        // ChannelPipeline holds all ChannelHandlers of channel
                        ch.pipeline().addLast(new EchoClientHandler());
                    }
                });
            // 7 Connect client to remote peer; wait until sync() completes connect completes
            ChannelFuture f = b.connect().sync();
            System.out.println(EchoClient.class.getName() + " connect on " + f.channel().localAddress());
            // #8 Wait until ClientChannel closes. This will block.
            f.channel().closeFuture().sync();
            System.out.println(EchoClient.class.getName() + " close");
        } finally {
            // #9 Shut down bootstrap and thread pools; release all resources
            group.shutdownGracefully().sync();
            System.out.println(EchoClient.class.getName() + " finally");
        }
    }
 
    public static void main(String[] args) throws Exception {
        /*
        if (args.length != 2) {
            System.err.println("Usage: " + EchoClient.class.getSimpleName() + " <host> <port>");
            return;
        }
        // Parse options.
        final String host = args[0];
        final int port = Integer.parseInt(args[1]);
        */
        new EchoClient("127.0.0.1", 8888).start();
    }
}
```

和服务器一样，NIO Transport 被使用。其实，使用不同的 transport 没有关系，你可以同时在服务端和客户端使用不同的 Transport。比如，在服务端使用 NIO Transport，在客户端使用 OIO Transport 也是可以的。在第四章中，你会学习到更多在什么场景应该用什么 Transport 的知识。

我们来强调一下本节的关键点：



- 一个 Bootstrap 实例被创建去启动客户端。
- 一个 NioEventLoopGroup 实例被创建，并被分配去处理事件，像创建新连接，接收数据，发送数据，等等。
- 一个远程的，客户端将要连接的 InetSocketAddress 实例被配置。
- 一个处理器被设置，一单连接被建立，它就会执行。
- 所有的工作被配置好后，Bootstrap.connect() 方法被调用去连接远程服务端（我们上一节创建的 EchoServer ）。



#### 2.4.2 实现客户端逻辑



我将会继续这个例程简单，所有用到的类，都会在接下来的章节中详细讨论。

```

package chapter2;
 
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandler.Sharable;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.util.CharsetUtil;
 
/**
 * EchoClient业务逻辑处理程序，原书的例程中，这个代码有一些Bug，我已经修复
 */
// #1 Annotate with @Sharable as it can be shared between channels
@Sharable
public class EchoClientHandler extends SimpleChannelInboundHandler<ByteBuf> {
    // 原书 bug fix : https://forums.manning.com/posts/list/33237.page
    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        // #2 Write message now that channel is connected
        ctx.write(Unpooled.copiedBuffer("Netty rocks!", CharsetUtil.UTF_8));
        ctx.flush();
        System.out.println("Client send:" + "Netty rocks!");
    }
 
    @Override
    public void channelRead0(ChannelHandlerContext ctx, ByteBuf in) {
        // #3 Log received message as hexdump
        System.out.println("Client received:" + in.toString(CharsetUtil.UTF_8));
    }
 
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        // #4 Log exception and close channel
        cause.printStackTrace();
        ctx.close();
    }
}
```

### 2.5 编译运行Echo程序客户端和服务端



2.5.1 编译客户端和服务端

2.5.2 运行客户端和服务端



### 2.6 总结



本章通过一个基础的客户端和服务端通讯程序，对Netty有了初步了解。你学习了怎么编译本书例子中的代码，以及安装需要的工具。这些工具会被本书接下来的更高级的例子使用。

本章也通过例子，演示了用Netty进行开发的时候可能发生的故障。最后，你学习了如何拦截和处理异常。

# 后记

