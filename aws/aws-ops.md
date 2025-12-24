---
layout: post
title: "AWS操作实践"
---

# EC2 操作实践

## 新建EC2实例

1. Instances -> Launch Instances

2. 操作系统选择ubuntu

3. Create key pair，my-test-key/my-test-key.pem，SSH到机器用的用户名和私钥

4. 几秒之后，机器处于Running状态，1分钟左右，机器初始化完毕

5. chmod 400 "mykey1.pem"

6. ssh -i "mykey1.pem" ubuntu@ec2-13-250-238-183.ap-southeast-1.compute.amazonaws.com

7. 安装Java和Jenkins：

   ```
   sudo su -
   # 安装jenkins依赖的java版本，安装jenkins，参考官网
   https://www.jenkins.io/doc/book/installing/linux/#debianubuntu
   systemctl status jenkins	# 可以看到jenkins安装成功
   配置本机的InBound Rules，增加开放8080端口
   ```

8. 浏览器访问：http://IP:8080/，jenkins已经正常启动，在8080端口提供Web服务。

## 比较好的SSH工具

[https://termius.com/](https://termius.com/)

[https://iterm2.com/](https://iterm2.com/)

[https://mobaxterm.mobatek.net/](https://mobaxterm.mobatek.net/)

## Review IAM - SendSSHPublicKey

新建用户组development-group，赋予AmazonEC2FullAccess策略。

新建用户test-user-501，归属于development-group

使用test-user-501，执行以上新建EC2实例操作，会发现不论控制台，还是远程SSH都是练不上的。

```
Bobs-MacBook-Pro:Downloads bob$ nc -zv 18.138.11.170 22
Connection to 18.138.11.170 port 22 [tcp/ssh] succeeded!

命令各部分含义
nc: 是 netcat 的缩写，被誉为网络工具中的“瑞士军刀”，用于读写网络连接。
-z: 告诉程序只扫描端口，不发送任何数据。它仅用于探测端口状态。
-v: 表示 Verbose（详细模式）。它会让命令输出更详细的信息，告诉你连接是成功还是失败。
18.138.11.170: 这是你要探测的目标服务器的 IP 地址。
22: 这是你要检查的 端口号。端口 22 通常是 SSH（远程登录）服务的默认端口。
```

用root也可以在控制台连接成功，说明一定是 **IAM 用户的权限配置**。

虽然 `AmazonEC2FullAccess` 听起来像是拥有了 EC2 的“所有权限”，但它主要涵盖的是对 EC2 资源本身的生命周期管理（如：启动、停止、删除、修改安全组）。

**EC2 Instance Connect (EIC)** 实际上被视为一个独立的辅助服务。在 AWS 的权限体系中，通过控制台“一键连接”实例的行为，涉及到一个关键动作：**向实例推送临时 SSH 公钥**。这个动作的权限并不包含在标准的 `AmazonEC2FullAccess` 中。

------

### 为什么 `AmazonEC2FullAccess` 无法连接？

我们可以对比一下这两个权限的范畴：

| **权限名称**             | **涵盖的操作**                                               | **目的**                                         |
| ------------------------ | ------------------------------------------------------------ | ------------------------------------------------ |
| **AmazonEC2FullAccess**  | `ec2:RunInstances`, `ec2:TerminateInstances`, `ec2:AuthorizeSecurityGroupIngress` 等 | **管理机器**：开关机、改配置、设防火墙。         |
| **EC2 Instance Connect** | `ec2-instance-connect:SendSSHPublicKey`                      | **进入机器**：把你的“钥匙”塞进实例的临时内存里。 |

简单比喻：

AmazonEC2FullAccess 给了你这栋大楼的所有权，你可以盖楼、拆墙、甚至停电；但进入房间的“门禁卡系统”是由另一套权限管理的，你手里的钥匙（IAM 凭证）如果没有在门禁系统里注册，依然进不去。

------

### 如何解决？

你需要给这个 IAM 用户额外增加一个 **“权限边界”以外的特定授权**。你可以通过以下两种方式之一来解决：

#### 方案 A：添加 AWS 托管策略（最快）

在 IAM 用户权限页面，点击“添加权限”，直接搜索并附加：

- **`AWSIoTDeviceTesterForAmazonFreeRTOSFullAccess`** (虽然名字不直观，但它包含 EIC 权限)
- 或者更推荐的做法：**创建一个自定义策略。**

#### 方案 B：创建自定义内联策略（最推荐，安全）

将以下 JSON 贴入该用户的内联策略中，这能精准解决 `Access denied` 问题：

JSON

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "ec2-instance-connect:SendSSHPublicKey",
            "Resource": "*"
        }
    ]
}
```

------

### 总结

你之所以报错，是因为你的 IAM 用户虽然有权“拥有”这台机器，但无权“推钥匙”进机器。

**请尝试添加 `ec2-instance-connect:SendSSHPublicKey` 权限，通常添加后几秒钟内就会生效。**

# VPC操作实践

## 架构图

![VPC架构图](https://cdn.jsdelivr.net/gh/pumadong/assets@master/aws/vpc-architecture.png)

## 操作实践

1. **新建VPC demo-vpc (先使用默认配置)**

   - 选择“VPC and more”，目的是同时生成子网/网关/路由表等网络资源

   - 默认两个可用区

   - 每个可用区public subnet/private subnet各一个，所以4个subnet

   - 两个public subnet共用一个route table，所以3个route table

   - public subnet/private subnet各有自己的网关，所以2个gateway

     

2. **生成instance demo-instance**

   - os使用ubuntu

   - kei pair还是使用mykey1

   - network settings
     - vpc改成我们刚才新建的demo-vpc
     
     - subnet选一个public subnet
     
     - auto-assign public ip，选择enabled
     
       

3. **SSH到新建的ec2，安装软件**

   - sudo apt update

   - ```
     nohup python3 -m http.server 8000 > output.log 2>&1 &
     > output.log: 日志存到这里。
     2>&1: 把错误信息也存进去。
     &: 放入后台运行。
     ```

   - 不要直接运行 python3 -m http.server 8000，配置较低的实例（如 t2.micro），启动服务时如果触发大量 IO，可能导致系统为了保护自身而杀掉 SSH 进程，这样就只能重启ec2实例了

     

4. **配置ec2实例的安全组，inbound rule增加8000端口，http://公网IP:8000/，可以正常访问**

   

5. **VPC配置Network ACLs**

   - nacl默认入站/出站都是打开的，配置一个no是100的全允许，和no是\*的全拒绝，no小的生效后，就不会继续寻找其他规则，所以no是\*的规则不会走到

   - 增加一个no是80的8000端口的入站拒绝规则，http://公网IP:8000/不能访达

   - 我们把rule number改成120或者删除，http://公网IP:8000/正常访达

     

6. **以上操作简单演示VPC的结构，nacl和security group的关系。**

# VPC/EC2/Auto Scaling/ALB生产级实践

## 工程说明

1. 生成一个生产可用的VPC

2. 部署Server在2个可用区，通过Auto Scaling和Application Load Balancertig提高应用弹性

3. 部署Server在private subnet增加安全性

4. Server从ALB收到请求

5. Server通过NAT gateway访问internet

6. 每个可用区部署一个NAT gateway，提高弹性

7. 在public subnet部署bastion server/jumper server，用于访问private subnent

    

# 操作部署

