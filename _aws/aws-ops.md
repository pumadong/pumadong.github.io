---
layout: post
title: "AWS操作实践"
date: 2025-12-22 18:30:00 +0800  # 标准格式
description: "记录AWS各个服务的操作实践，最终形成生产可用的SOP操作模版。"
---

# 一、EC2 操作实践

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

# 二、VPC操作实践

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

   - nohup python3 -m http.server 8000 > output.log 2>&1 &
   
     \> output.log: 日志存到这里。
   
     2>&1: 把错误信息也存进去。
   
     &: 放入后台运行。
   
   - 不要直接运行 python3 -m http.server 8000，配置较低的实例（如 t2.micro），启动服务时如果触发大量 IO，可能导致系统为了保护自身而杀掉 SSH 进程，这样就只能重启ec2实例了
   
     
   
4. **配置ec2实例的安全组，inbound rule增加8000端口，http://公网IP:8000/，可以正常访问**

   

5. **VPC配置Network ACLs**

   - nacl默认入站/出站都是打开的，配置一个no是100的全允许，和no是\*的全拒绝，no小的生效后，就不会继续寻找其他规则，所以no是\*的规则不会走到

   - 增加一个no是80的8000端口的入站拒绝规则，http://公网IP:8000/不能访达

   - 我们把rule number改成120或者删除，http://公网IP:8000/正常访达

     

6. **以上操作简单演示VPC的结构，nacl和security group的关系。**

# 三、VPC/EC2/Auto Scaling/ALB生产级实践

## 架构图

![架构图](https://cdn.jsdelivr.net/gh/pumadong/assets@master/aws/vpc-example-private-subnets.png)

## 工程说明

1. 生成一个生产可用的VPC

2. 部署Server在2个可用区，通过Auto Scaling和Application Load Balancertig提高故障恢复能力

3. 部署Server在private subnet增加安全性

4. Server从ALB收到请求

5. Server通过NAT gateway访问internet

6. 在两个可用区分别部署NAT gateway，提高故障恢复能力

7. 在public subnet部署bastion server/jumper server，用于访问private subnent

    

## 工程实现

### Create VPC

1. **Name：**aas-prod-example

2. **NAT gateways($)：**Zonal，1 per AZ

3. **VPC endpoints：**none

   

### Create Auto Scaling group

1. **先生成Launch template**

   1. **Name：**aws-prod-example

   2. **OS：** Ubuntu

   3. **Instance type：** t2.micro(Free tier eligible)

   4. **Key pair(login)：** mykey1

   5. **Create secirity group：**使用我们上面新建的VPC，开发22/8000端口

      

2. **根据刚才生成的template新建Auto Scaling group**

   1. **Name：**aws-prod-example

   2. **选择刚才创建的template**，next

   3. **选择VPC：**上面创建的VPC

   4. **subnets：** 两个private子网，因为我们是要对应用部署所在的private subnet自动伸缩，next

   5. **不需要load balancing**，我们是在public subnet进行lb，next

   6. **Desired capacity：**2，min选择1，max选择4，next

   7. **Add notifications - optional**，next，**Add tags - optional**，next

   8. **两个ec2主机新建完毕**

      

### 生成bastion server/jumper server

1. **Name：** bastion-host

2. **OS：** Ubuntu

3. **Instance type：** t2.micro(Free tier eligible)

4. **Key pair(login)：** mykey1

5. **Network settings**

   1. **选择VPC：**上面创建的VPC

   2. **Subnet：**选择public subnet

   3. **Auto-assign public IP：**Enabled

      

### 通过跳板机到两个ec2实例安装应用

1. **登录跳板机：**ssh -i "mykey1.pem" ubuntu@ec2-54-169-19-82.ap-southeast-1.compute.amazonaws.com

2. **复制本地的秘钥到跳板机：**scp -i mykey1.pem mykey1.pem ubuntu@ec2-54-169-19-82.ap-southeast-1.compute.amazonaws.com:/home/ubuntu

3. **到第一个ec2实例部署应用**
   1. ssh到第一个跳板机
   
   2. echo 'hello world, server 1' > index.html
   
   3. nohup python3 -m http.server 8000 > output.log 2>&1 &
   
      
   
4. **到第二个ec2实例部署应用**
   1. ssh到第二个跳板机
   
   2. echo 'hello world, server 2' > index.html
   
   3. nohup python3 -m http.server 8000 > output.log 2>&1 &
   
   4. 我们也可以不部署第二个ec2实例，可以看到下面将要配置的ALB是有health check的，此时会将所有流量打到第一个ec2实列
   
      

### Create Application Load Balancer

1. **Name：**aws-prod-example

2. **选择VPC：**上面创建的VPC

3. **可用区和子网：**选择至少2个AZ，每个AZ选择一个public subnet

4. **选择Security groups：** 选择我们新建ACG时新建的

5. **Listeners and routing：**
   1. Listner：Port用默认80

   2. 新建Target group，并选中
      1. 端口8000，next
      
      2. 选择ec2实例，Include as pending below，next
      
         

6. **报”Not reachable“错误**

   1. Security，在Security group增加80端口

   

7. **浏览器访问ALB的DNS name**

   1. http://aws-prod-example-759217952.ap-southeast-1.elb.amazonaws.com/

   2. 可以看到流量均匀的打到两台server

      

# 四、S3操作实践

## 演示设置访问权限

1. 使用账户A，创建S3 bucket

2. Name：app1-payments-prod-examp-bob-dong.com，这个名称在AWS全局唯一

3. 使用账号B，没有S3相关权限，则查看s3 bucket列表，生成bucket，都无法进行

4. 授予账户B，AmazonS3FullAccess，则查看s3 bucket列表，生成bucket，都正常进行

5. 场景限制：即使对方有S3访问权限，也不能访问我的Bucket

   Permission -> Bucket Policy -> Edit

   ```
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Sid": "only-team-x-can-access",
               "Effect": "Deny",
               "Principal": "*",
               "Action": "s3:*",
               "Resource": "arn:aws:s3:::app1-payments-prod-examp-bob-dong.com",
               "Condition": {
                   "StringNotEquals": {
                       "aws:PrincipalArn": "arn:aws:iam::275695461302:root"
                   }
               }
           }
       ]
   }
   ```

6. 账户B可以在列表中看到Bucket，点击进入看不到内容了：**Insufficient permissions to list objects**

   

## 演示托管静态站点

1. 创建S3 bucket，Name：app2-payments-prod-examp-bob-dong.com

2. 修改：Block public access，设置为on

3. 创建成功后，到Properties页，修改 Static website hosting设置

4. 设置Bucket Policy，让互联网上的每个人都能访问

   ```
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Sid": "PublicReadGetObject",
               "Effect": "Allow",
               "Principal": "*",
               "Action": [
                   "s3:GetObject"
               ],
               "Resource": [
                   "arn:aws:s3:::app2-payments-prod-examp-bob-dong.com/*"
               ]
           }
       ]
   }
   ```

5. https://s3.ap-northeast-2.amazonaws.com/app2-payments-prod-examp-bob-dong.com/index.html，访达。

   

# 五、CLI操作实践

到现在为止，都是通过Brower操作AWS提供的各种服务，看着浏览器经常的龟速刷新，是不是在寻求解决办法？

所以，Command Line来了，因为快速、直接，是必不可少的快捷操作工具^_^

## 安装

**[https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)**

## 配置访问密钥（Access Keys）

1. 右上角点击用户名，选择Security Credentials

2. **Access keys** 新建访问密钥，如果此IAM用户被拒绝访问，先授权。

   赋予用户能**查看、创建和删除自己的 Access Keys**的权限。

   ```
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Sid": "AllowManageOwnAccessKeys",
               "Effect": "Allow",
               "Action": [
                   "iam:CreateAccessKey",
                   "iam:DeleteAccessKey",
                   "iam:ListAccessKeys",
                   "iam:UpdateAccessKey",
                   "iam:GetAccessKeyLastUsed"
               ],
               "Resource": "arn:aws:iam::*:user/${aws:username}"
           }
       ]
   }
   ```

   #### 为什么必须指定变量 `${aws:username}`？

   使用 `${aws:username}` 变量是一个**安全最佳实践**。它能确保：

   1. **权限自适应**：你把这个策略丢进一个用户组（Group）里，组内所有成员都只能看到“自己的”密钥。
   2. **横向隔离**：User A 即使有了这个权限，也无法通过 API 看到 User B 的密钥列表，从而防止了权限提升攻击。

## 本地配置：aws configure

![AWS CLI Configure](https://cdn.jsdelivr.net/gh/pumadong/assets@master/aws/aws-configure.png)

## 运行命令



**操作的所有命令都是和当前的regin：ap-southeast-1绑定的**



**列出s3 bucket：**

```
aws s3 ls
```

**一键显示所有 Bucket 及其文件（Shell 循环）**

```
for bucket in $(aws s3 ls | awk '{print $3}'); do 
    echo "--- Bucket: $bucket ---"; 
    aws s3 ls s3://$bucket --recursive --human-readable --summarize; 
    echo ""; 
done
```

**列出子网：**

```
aws ec2 describe-subnets --query 'Subnets[*].{SubnetID:SubnetId, AvailabilityZone:AvailabilityZone, CIDR:CidrBlock}' --output table
```

**列出子网并显示是否为public：**

一个子网是 Public 还是 Private，**唯一决定因素**是它的路由表中是否存在一条指向 **Internet Gateway (IGW)** 的路由。

```
aws ec2 describe-subnets --query 'Subnets[*].[SubnetId, VpcId, CidrBlock]' --output text | while read subnet vpc cidr; do
    # 1. 查找直接关联的路由表，如果没有，则查找 VPC 的主路由表
    rtb=$(aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=$subnet" --query 'RouteTables[0].RouteTableId' --output text)
    if [ "$rtb" == "None" ]; then
        rtb=$(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$vpc" "Name=association.main,Values=true" --query 'RouteTables[0].RouteTableId' --output text)
    fi

    # 2. 检查该路由表中是否有目的地为 0.0.0.0/0 且 Target 为 igw-xxx 的路由
    is_public=$(aws ec2 describe-route-tables --route-table-ids "$rtb" --query 'RouteTables[0].Routes[?GatewayId && starts_with(GatewayId, `igw-`)].GatewayId' --output text)

    # 3. 格式化输出
    type="Private"
    if [ ! -z "$is_public" ]; then type="Public"; fi

    printf "%-20s | %-20s | %-18s | %-15s | %-10s\n" "$subnet" "$vpc" "$rtb" "$cidr" "$type"
done
```

**列出安全组：**

```
aws ec2 describe-security-groups --query 'SecurityGroups[*].{Name:GroupName, ID:GroupId, VPC:VpcId, Description:Description}' --output table
```

**生成EC2实例：**

```
aws ec2 run-instances --image-id ami-00d8fc944fb171e29 --instance-type t3.micro --key-name mykey1 --subnet-id subnet-02313ca720be6bdc7 --security-group-ids sg-0cc7113407dc17567 --tag-specifications 'ResourceType=instance, Tags=[{Key=Name, Value=MyInstanceCreatedByCli}]'
```

**注意事项：**

-  **选用的image要和instance type支持的架构类型一致**，这帮你理解机器架构和镜像架构需要一致。
- **subnet和security group要属于同一个VPC网络**，这帮你理解**Security Group（安全组）** 是在 VPC 级别定义的。它像一个分布式防火墙，虽然最终“作用”在 EC2 的网卡上，但它的**归属权**属于 VPC。

1. 官方命令参考

   [https://docs.aws.amazon.com/cli/latest/reference/](https://docs.aws.amazon.com/cli/latest/reference/)

