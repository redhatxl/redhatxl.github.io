# 3.实战Gitlab CI + Supervisor 发布Python项目

## 一 背景

在上一节我们系统的了解了Gitlab CI的流程及如果将一个项目集成CI的相关知识，在这一节课我们来通过实战Gitlab CI在Python项目中的应用，进一步领略Gitlab CI的魅力。

本次项目为一个Python的项目，Python项目为们都知道不需要去编译和打包。我们通过这一节系统来从Gitlab Runner的注册到`gitlab-ci.yml`文件的编写再到测试部署上线，来完整的实战Pyhon项目的CI。

## 二 思路解析

我们都知道Python为解释性语言，Python 解释型语言不需要编译打包，自身源代码即目标部署文件，但是在部署时需要注意一些实现

### 2.1 环境管理

#### 2.1.1 环境问题

如果我们有多个Python项目，在这些项目的构建中我们需要运行单元测试，但是这些项目的Python版本不一致，依赖的库的版本也不一致此时我们该怎么解决这样的部署问题呢？

#### 2.1.2 解决方法

* 如果我们为了节省服务器资源，多个项目共用一个公共的gitlab-runner用这台服务器来进行任务构建发布，此时针对Python不一致的问题，我们可以利用Python环境管理工具例如anconda/virtulenv等来进行每个项目一个独立的环境，对其进行每次job任务的部署。
* 当然如果服务器资源充足，那们我们对于Python项目，可以在部署的目标服务器安装gitlab-runner，将其注册在项目下，直接利用其进行单环境Job。

### 2.2 进程管理

#### 2.2.1 进程问题

在部署目前主力的Python框架项目，例如Django/Flask/Tornado等，如将项目运行在前台或利用`nohup &`放在后台，gitlab pipeline无法进行退出且后期不便于我们维护管理，那我们该如果更加优雅的管理启动进程呢？

#### 2.2.2  解决方法

* 对此，有的小伙伴可能想到，很简单我们可以通过编写启动脚本，但是利用脚本启动进程，不仅编写脚本耗时耗力且需要做单独对进程监控，不便于我们统一管理维护。

* 我们可以利用进程管理系统Superviosr解决此痛点，Superviosr为用Python语言开发对一套通用进程管理系统，可利用pip或yum进行安装，其能将一个普通对命令行进程变为daemon，并监控其进程状态，可通过配置如果监控进程异常退出则自动对其进行重启，同时也拥有web管理界面方便管理查看。来实现对部署项目start/stop/restart/reload服务管理，通过fork/exec的方式把这些被管理的进程，当supervisor的子进程来启动，完美解决来项目部署对难题。

### 2.3 软件集成

#### 2.3.1 软件如何集成

我们知道Jenkins作为持续集成的利器，其一大特点就是丰富的插件，那么我们使用Gitlab CI该如何集成其他软件呢。

#### 2.3.2 解决方法

* 在Gitlab CI中，如果我们有使用到其他的工具或软件，可以在注册的gitlab-runner上面进行客户端或工具的安装，之后在CI的scripts中具体来调用gitlab-runner上面的工具来实现我们的具体需求即可。
* 既然Gitlab CI集成其他软件是在gitlab-runner上面，那么如果有需求其他系统调用Gitlab的接口，或使用gitlab的功能该怎么办呢，gitlab提供了新建应用，可以对应用授予特定权限，接着创建token来供其他应用调用。

## 三 架构解析

在我们解决了上面的一些疑惑，已经对部署有了整体的思路之后，我们来进行具体的实施操作

### 3.1 服务器列表

|         名称         |      IP      |         软件         |          备注           |
| :------------------: | :----------: | :------------------: | :---------------------: |
|    Gitlab-Server     | 10.57.61.253 |    gitlab-server     |      Gitlab 服务器      |
| Gitlab-Common-Runner | 10.57.61.10  |    gitlab-runner     | 公用gitlab-runner服务器 |
|     Des-Server01     | 172.16.100.2 | miniconda/supervisor |   项目部署目标服务器    |

### 3.2 架构图

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200222215813.png)

如上图所示，我们的gitlab-runner可以有两种模式

* 一种为公共的gitlab-runner：
  * 优点：节省服务器资源，在一台服务器部署gitlab-runner，免去在每个部署的服务器都需要安装维护应用。
  * 缺点：环境管理复杂，需要环境管理软件配合使用，如果此服务器的gitlab-runner挂掉，那其他的任务也会收到影响，并且这台服务器服务器需要与部署的目标服务器能够进行ssh免密钥远程登录。
* 每台部署的目标服务器都安装gitlab-runner：
  * 优点：每个服务器环境都只维护自己的环境，不需要管其他的，如果其他gitlab-runner异常不会造成其他影响
  * 缺点：安装繁琐，每天部署的目标服务器都需要安装并后期维护，如果gitlab-runne众多会对服务端造成压力。

那们对于我们该进行如何取舍呢，在这里大家可以根据自己的业务及每个公司不同业务线的规模来衡量，例如我们可以将一个项目的多条部署任务共用一个公共的gitlab-runner，然后对于其他单独的少量的直接单独安装gitlab-runner注册到服务端。

闲话不多说，我们来进行实操，

## 四 安装部署

### 4.1 前期准备

- 具有Gitlab Server

- Gitlab-runner服务器的gitlab-runner用户可以免密钥登录des-server服务
- 在gitlab-common服务及目标服务器安装配置conda虚拟环境
- 在目标服务器进行supervisor部署配置

### 4.2 gitlab-runner安装配置

#### 4.2.1 gitlab-runner安装

在gitlab-common服务器上安装gitlab-runner

```shell
# 配置yum源
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-ci-multi-runner/script.rpm.sh | sudo bash
# 安装runner
sudo yum install -y gitlab-ci-multi-runner
```

#### 4.2.2 项目权限配置

需要项目开启pipeline的权限，需要手动开启 settings->General->Visibility, project features, permissions->Pipelines ，可以配置所有人或此项目的Members可以配置管理Pipeline。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200222221640.png)

#### 4.2.3 记录注册信息

在此需要集成Gitlab CI的项目中，记录注册Runner的URL和token，gitlab 上的settings->CI/CD>Runners

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200222222334.png)

#### 4.2.4 服务器进行注册

在gitlab-commo-runner服务器进行gitlab-runner的注册,此处我们选择的Executor为shell，注意在此注册中鞋的tags在我们后面的`.gitlab-ci.yml`中将会用到，用来指定使用那个runner来运行此job。

```shell
# gitlab runner注册到平台
sudo gitlab-ci-multi-runner register
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200222222321.png)

在我们注册完成后，我们可以登录gitlab的项目下去查看已经注册上来的Runner，在此我们需要注重记录我们的tags，后续在文件中引用。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200222223655.png)

至此我们已经成功将gitlab-runner注册到gitlab server上，接下来让我们来实际编写项目的部署文件

### 4.3 CI文件的编写

#### 4.2.1 .gitlab-ci.yml

在项目中集成`.gitlab-ci.yml`

```yaml
stages:
  - clean_env            # 清理环境及杀死进程
  - deploy_src           # 部署源码
  - install_dependency   # 更新依赖  
  - restart_server       # 重启服务
  - check_server         # 检测服务
  

variables:							# 定义变量
  BASE_DIR: "/data/go2cloud_platform/"

job clean_env_job:			# 清理环境job
  stage: clean_env
  script:
    - ssh -o stricthostkeychecking=no root@172.16.100.2 pkill supervisord || true
    - ssh -o stricthostkeychecking=no root@172.16.100.2 killall /data/miniconda3/bin/python || true
    - ssh -o stricthostkeychecking=no root@172.16.100.2 killall /data/miniconda3/envs/go2cloud_platform/bin/python || true
    - ssh -o stricthostkeychecking=no root@172.16.100.2 rm -rf /project${BASE_DIR}*
  tags:
    - go2cloud_platform			# 注册在gitlab 上的gitlab runner的tag
  only:
    - dev
  when: always


job deploy_src_job:
  stage: deploy_src					# 将项目源码拷贝到des服务器 
  script:
    - scp -r /home/gitlab-runner/builds/QFafrHEq/0/devops/${BASE_DIR}* root@172.16.100.2:/project${BASE_DIR}
    - ssh -o stricthostkeychecking=no root@172.16.100.2 cp /project/config/config.yml /project${BASE_DIR}
  tags:
    - go2cloud_platform
  only:
    - dev
  when: always


job install_dependency_job:
  stage: install_dependency			# 安装依赖
  script:
    - ssh -o stricthostkeychecking=no root@172.16.100.2 /data/miniconda3/envs/go2cloud_platform/bin/python -m pip install -r /project${BASE_DIR}requirements/requirements.txt
  tags:
    - go2cloud_platform
  only:
    - dev	
  when: manual									# 为手动运行


job restart_server_job:					# 启动进程
  stage: restart_server
  script:
    - ssh -o stricthostkeychecking=no root@172.16.100.2 sleep 7
    - ssh -o stricthostkeychecking=no root@172.16.100.2 supervisord -c /etc/supervisord.conf
    - ssh -o stricthostkeychecking=no root@172.16.100.2 ps -ef |grep supervisord |grep -v grep
  tags:
    - go2cloud_platform
  only:
    - dev
  when: always

job check_server_job:					  # 检查服务
  stage: check_server
  script:
    - ssh -o stricthostkeychecking=no root@172.16.100.2 sleep 7
    - ssh -o stricthostkeychecking=no root@172.16.100.2 ps -ef|grep "8088"
  tags:
    - go2cloud_platform
  only:
    - dev
  when: always

```

#### 4.3.2 Stages详解

在此项目中我们使用的common-server，整个pipeline分为五个stage

- 清理环境：可以配置为全部删除目标源代码或这rsync/scp增量覆盖到目标服务器，如果增量部署，需要考虑迁移数据库到重复执行。
- 部署源码：将gitlab-runner服务器pull下来到代码scp到目标服务器到目标服务器部署目录，
- 安装依赖：此处有可能后期更新requirements，配置为manual手动去执行更新，如果我们知道我们这次有新的依赖包安装新增或更新，手动去安装。
- 重启服务：启动为supervisord服务，启动后可以自动启动配置到项目服务。
- 检测进程：为了更直观检测运行的进程，为们在最后检测进程判断部署是否成功。

## 五 测试

当我们提交代码后，merge到我们在CI文件中写的dev分支后，我们可以看到我们的pipeline的运行状态。

### 5.1 查看运行结果

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200222225038.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200222224755.png)

### 5.2 查看某个具体job

查看具体jobs信息，在右上角如果部署有异常，我们可以点击Retry进行重试，在页面中可以看到实时的日志信息。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200222225227.png)

### 5.3 查看CI/CD charts

charts直观的展示来构建到成功及失败图表，一般如果我们CI文件经过调试稳定以后，后续就不用关心上线问题。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200223211112.png)

### 5.4 构建通知

Gitlab CI的构建无需我们Jenkins一样去进行配置，如果Gitlab 已经配置了邮件通知，那们我们在执行完pipeline后会改项目相关如人员发送构建通知。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200223211140.png)



### 5.5 日志查看

在构建完成后，我们可以详细看我们构建的每一步日志

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200223212005.png)

### 5.6 进程查看

为们可以登录服务器去查看进程，同时也可以利用Supervisor来查看进程

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200223211226.png)



### 5.7 CI徽章配置

为们在项目下面可以集成CI的徽章方便查看Pipeline的执行状态

Settings-> CI/CD->General pipelines -> Pipeline status

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200226203356.png)

在项目下进行集成

Settings->Badges

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200226203450.png)

## 六 注意事项

* 该项目我们需要提前准备好common-server服务器，需要在其上安装一些需要的环境；
* 在执行的时候可能前面几次都是进行测试，如果失败可以进入控制台详细查看日志；
* 在common-server上执行的用户为gitlab-runner用户，如果一些工具或命令没有权限，可以将该用户配置sudo权限。

## 七 反思

通过该项目为们为部署Python项目，直接部署源代形式，没有编译打包等过程，在其中就是利用了将日常每次部署使用的命令，进行组织编排成yaml的stage，同时利用了已经事先安装在common-server或des-server上的管理关键进行项目的环境和进程管理，我们可以根据自己的项目实际情况来进行stage规划，例如一个类型的项目共用一个gitlab-runner，或者对环境没有要求的可以共用一个runner， 如果仅是零散的单个项目可以直接将gitlab-runner部署在目标服务器上，在此抛砖引玉，读者可以根据自己的实际情况来考量修改。

​		