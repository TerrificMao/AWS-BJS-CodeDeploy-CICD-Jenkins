## **AWS-BJS-CodeDeploy-CICD-Jenkins**

## 目的

AWS DevOps - 配合Jenkins和CodeDeploy实现代码自动化部署

作为DevOps和微服务的深入践行者，Amazon在内部积累了许多持续集成、交付和部署的自动化工具和平台。本文主要介绍如何在AWS云上，使用CodeDeploy，并配合Jenkins来构建持续集成/持续交付的管道，自动化代码部署和版本迭代。在查看本文之前，建议大家先阅读一下AWS架构师代闻老师写的关于CodeDeploy的文章。

<https://aws.amazon.com/cn/blogs/china/getting-started-with-codedeploy/>  

自动化部署解决方案

![img](assets/AWS-DevOps.png)

## 一、创建EC2实例并安装CodeDeploy Agent

创建Amazon EC2实例，选择实例类型和添加存储。

![img](assets/01.png)

在“高级详细信息”里面输入启动脚本

![img](assets/02.png)

```bash
#!/bin/bash
yum -y update
yum install -y ruby
yum install -y aws-cli
cd /home/ec2-user
aws s3 cp s3://aws-codedeploy-cn-north-1/latest/install . --region cn-north-1
chmod +x ./install
./install auto
```

EC2启动成功后，使用SSH到该EC2，使用如下命令检验Agent是否工作正常。

```bash
sudo service codedeploy-agent status
Result: The AWS CodeDeploy agent is running as PID 3523
```

## 二、创建应用程序负载均均衡ALB

![img](assets/03.png)

创建Target Group

![img](assets/04.png)

 

## 三、创建CodeDeploy环境

点击“创建应用程序” 

![img](assets/05.png)

输入应用程序名称，和部署组的名称。CodeDeoploy支持两种部署方式，“就地部署”和“蓝绿部署”，更多关于部署类型请参考:

<https://docs.aws.amazon.com/zh_cn/codedeploy/latest/userguide/applications-create-in-place.html>

环境配置可以选择Auto Scaling组、Amazon EC2 实例，或者本地实例。

![img](assets/06.png)

该文档使用的是Amazon EC2实例方式，通过Tag指标相对应的EC2实例。

![img](assets/07.png)

 启用负载均衡，选择之前创建的Target Group。

![img](assets/08.png)配置回滚的策略，比如选择“部署失败时回滚”。可以根据情况添加触发器，并订阅Amazon SNS进消息通知，也可以配置CloudWatch警报，当指标超过或者低于CloudWatch中设定的阈值就可以自动触发或者停止部署。

![img](assets/09.png)CodeDeploy环境创建成功

![img](assets/10.png)

##  四、使用Github托管源代码，并配置webhook自动触发

首先进入自己的Github地址，点击<https://github.com/settings/tokens>，生成GitHub token，这个token用于jenkins访问GitHub。

![img](assets/11.png)

为需要做CI/CD的GitHub创建hook，实现代码更新自动通知Jenkins，Payload URL设置Jenkins Server的地址，默认Jenkins监听8080端口。记录下生成的token字符串，比如： `bf6adc27311a39ad0b5c9a63xxxxxxxxxxxxxx`

创建一个新的GitHub Repository

![img](assets/12.png)

创建本次环境所需要的Git仓库，比如名为AWS-BJS-CodeDeploy-CICD-Jenkins。点击“Settings”配置webhooks。

![img](assets/13.png)

点击“Add Webhooks”

![img](assets/14.png)

在Payload URL，输入http://EC2公网IP地址/github-wekhook/，如下图所示：

![img](assets/15.png)

 

## 五、部署Jenkins，并安装CodeDeploy插件

安装如何脚本安装Jenkins，默认Java的环境是1.7的，可以先升级到Java 1.8版本。

```bash
sudo -s
java –version
yum install java-1.8.0
yum remove java-1.7.0-openjdk
wget -O /etc/yum.repos.d/jenkins.repo http://jenkins-ci.org/redhat/jenkins.repo
rpm --import http://pkg.jenkins-ci.org/redhat/jenkins-ci.org.key
yum install jenkins
chkconfig jenkins on
service jenkins start
//查看Jenkins默认密码
cat /var/lib/jenkins/secrets/initialAdminPassword
```

在浏览器输入输入EC2的公网IP地址（最好绑定一个弹性EIP），比如54.223.215.xx:8080，然后出现如下界面，输入上面得到的默认密码。

![img](assets/16.png)

![img](assets/17.png)

创建用户名和密码，就到如下界面，这个时候Jenkins就可以进行配置了。

![img](assets/18.png)

进入系统配置

![img](assets/19.png)

输入Jenkins URL，点击“Add”添加Jenkins

![img](assets/20.png)

输入Github获取的Access Token，点击“添加”。

![img](assets/21.png)

点击“Test Connection”，没有报错说明配置成功。

![img](assets/22.png)

添加管理插件

![img](assets/23.png)

添加AWS CodeDeploy的插件，点击“Install without restart”

![img](assets/24.png)

新建一Jenkins个项目，点击“Create a new project”

![img](assets/25.png)

配置Github项目的地址，源代码管理选择Git方式。

![img](assets/26.png)

触发构建，选择Github hook trigger for GITScm polling

![img](assets/27.png) 

选择“Post-build Actions”，输入CodeDeploy相关信息，区域选择AWS中国（北京）区域cn-north-1

![img](assets/28.png)

认证方式可以输入AWS Access Key和Security Key，如果是生产环境建议使用临时的credentials。

![img](assets/29.png)

**创建完成之后可以看到如下界面**

![img](assets/30.png)

## 六、测试

测试PHP代码下载地址

<https://github.com/TerrificMao/AWS-BJS-CodeDeploy-CICD-Jenkins> 

当代码部署成功时，输入EC2公网IP地址，可以看到如下界面，此时为V1版本。

![img](assets/31.png)

**此时，开发工程师在本地修改代码**

![img](assets/32.png)

Github本地客户端会自动识别到代码变更，输入描述内容以及关键词，点击“commit to master”，再点击右上方的“Sync”，代码就会自动推送到Github中。

![img](assets/33.png)

可以看到当代码提交到Github之后，Jenkins自动触发，拉取代码，进行构建。

![img](assets/34.png)

看到构建成功，自动打包，并把打完的Zip包自动上传到指定的S3存储桶上。

![img](assets/35.png)

打开CodeDeploy界面，发现自动触发了一次代码部署。

![img](assets/36.png)

部署正在进行中

![img](assets/37.png)

可以看到部署成功了，如果失败就回滚。

![img](assets/38.png)

 点击详情可以看到CodeDeploy自动部署的过程。

![img](assets/39.png)

刷新浏览器，发现代码自动化部署成功，替换成了V2版本。

![img](assets/40.png)

由于前面我们部署了负载均衡ALB，可以复制ALB的DNS名称到浏览器

![img](assets/41.png)

发现用负载均衡ALB访问也是成功的。

![img](assets/42.png)

上一台EC2在可用区b，刷新负载均衡器，会将流量自动分发到另一台机器上，此时在可用区a了。

![img](assets/43.png)

到此，一个比较简单的由ALB（负载均衡） + EC2（服务器） + Auto Scaling（自动扩展）组成架构，使用AWS CodeDeploy，再配合Jenkins可以实现代码的自动化部署和版本迭代。

![img](assets/44.png)

当然，AWS在DevOps方面提供的能力远远不止于此，AWS在开发、构建、测试、部署、搭建、监控、运维等各个维度都提供了托管的服务，可以让用户轻松完成持续部署、持续集成方面的自动化。

![img](assets/45.png)

最后，如果您想了解容器和Lambda方面的自动化部署方案，可以参考：

（1）基于Amazon EC2 Container Service的持续集成/持续交付解决方案

<https://aws.amazon.com/cn/blogs/china/continuous-integrationcontinuous-delivery-solution-based-on-aws-ecs/> 

（2）如何使用AWS CodePipeline，AWS CodeBuild与AWS CloudFormation实现Amazon ECS上的持续集成持续部署解决方案：

<https://aws.amazon.com/cn/blogs/china/how-to-implement-the-continuous-integrated-continuous-deployment-solution-on-amazon-ecs-using-aws-codepipelineaws-codebuild-and-aws-cloudformation/>

（3）AWS Lambda 配合 Jenkins 实现自动化持续部署

<https://aws.amazon.com/cn/blogs/china/aws-lambda-jenkins-automatically-deployment/> 

