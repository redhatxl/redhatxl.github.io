# 2.敏捷无敌之Gitlab CI架构流程详解

## 一 引子

在我们知道了Gitlab CI是什以及它能为我们带来什么，在我们利用Gitlab CI之前，需要先了解其架构流程，通过本节学习Gitlab CI的基础概念及相关元素，更加有利于之后的三篇实战理解，Gitlab CI基于自动执行脚本，最大限度的减少在环境部署及上线时引入错误的可能性，当一条Gitlab CI集成完成，从新代码的开发到部署，极少或者根本不需要人为的干预，在每次小的迭代中不断构建，测试和部署，尽可能早的试错，这种方式为最适合敏捷高效的开发。

那么如何开始我们的Gitlab CI之旅呢，其实你需要在你的项目中添加一个 .gitlab-ci.yml 文件，然后在组或者项目中注册一个 Runner，即可进行持续集成，那么什么又是Runner，`.gitlab-ci.yml`文件又该如何写呢，相信通过本章节，读者能清楚的了解Gitlab CI的架构流程及其元素组件，为我们后面的实战做好充分的准备。

## 二 基础概念

### 2.1 Gitlab-Runner

#### 2.1.1 Gitlab-Runer是什么

Gitlab CI的架构为C/S模型，Gtilab为服务端，其负责代码的托管，Gitlab CI任务的触发及结果日志记录等，真正运行我们在`.gitlab-ci.yml`中的定义的执行过程，是在客户端Gitlab-Runner上执行的，那么什么是Gitlab-Runner呢，其实就是一台运行有Gitlab-Runner的服务器或一个容器，只要安装有Gitlab-Runner，并且这个服务器在Gitlab Server端进行注册，这个Gitlab-Runner就可以在Gitlab Server端的`.gitlab-ci.yml`中使用此Runner来执行pipeline中的stage。

#### 2.1.2 Gitlab-Runer的分类

作为承载具体job的执行者，Runner可以分布在不同的服务器上，针对服务器同时一台主机也可以运行多个Runner

Gitlab-Runner的分类：

* Shared Runner：共享型Runner，所有的项目工程都可以利用此Runner来执行job。
* Specific Runner：特定型Runner，由于有权限控制此Runner只能为指定的项目工程服务。

- Group Runner：如果将Runner注册在项目组里面，那么该组内的项目可以指定公用这个Runner。
- Locked：Runner的一种状态，此刻该Runner以及处于Locked状态，被用项目锁定，不能用于其他项目构建
- Paused：Runner以及被暂停，不能接受任何Job任务。

了解了Runner的分类和状态，我们一起通过下图更直观的看下Runner

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200216225445.png)

在上图中，当研发人员提交代码或提交Merge Request时候触发Pipeline，然后通过Runner来执行，此时des-server可以安装部署多个Runner，同时也可以部署一个公共的Runner服务器，然后通过该公共服务器的Runner部署代码到des-server上。

#### 2.1.3 Executor

上面理解了Gitlab Runner的概念及分类，在我们将Gitlab Runner运行在服务器上后需要在平台进行注册该Gitlab Runner，注册的时候可以选择该Gitlab Runner的Executor，Gitlab CI为了实现在不同场景下运行构建的任务，分为很多的执行器。对于构建一个项目，不同的Executor支持不同的平台以及执行不同的方法，我们来了解各个Executor的适应场景，根据其特性并结合自己的业务来进行选择。

- Shell：Shell  Executor是配置的最简单的执行器，在一个服务器上安装Gitlab Runner后，直接将其注册到Server端口，并且后期运行的job所需的所有依赖项都需要手动安装在安装在这台注册的系统上。
- Docker：顾名思义，利用Docker 镜像来作为Executor，其能够保持干净的构建环境，并且容易进行依赖管理(构建项目的所有依赖项都可以放在 Docker 映像中)。
- Docker Machine and Docker Machine SSH（autoscaling）：Docker Machine 是 Docker 执行器的一个特殊版本，支持自动缩放。 它的工作原理与一般的 Docker 执行程序类似，但使用的是 Docker Machine 的构建机。
- VirtualBox：其为运行使用虚拟机作为执行job的方式，Gitlab CI提供两个系统虚拟化选项: VirtualBox 和 Parallels。 然后 GitLab Runner 连接到虚拟机并在其上运行构建版本，这种方式可以很大程度 降低基础设施成本。
- SSH：为了完整性，添加了 SSH 执行器，它是 GitLab Runner 连接到一个外部服务器，并在那里运行构建,类似一个跳板一样。
- Kubernete：在微服务盛行的今天，Kubernetes 执行器允许将构建过程放置在Kubernetes集群内完成， 执行者将调用 Kubernetes 集群 API，并为每个 GitLab 作业创建一个新的 Pod 来运行构建任务，构建完成后去释放掉次POD，当有请求在创建，此中动态方式非常利用资源的利用以及环境一致性。

### 2.2 .gitlab-ci.yml

了解了gitlab-runner，接下来我们需要了解`.gitlab-ci.yml`中的个元素概念，了解了这些，后期通过这些基础配置，来完成复杂高级的CI任务。

#### 2.2.1 Pipeline

Pipeline：为一个构建任务的集合，其为一次构建任务，当开发人员提交代码或Merge Request合并的时候都可以触发Pipeline，其内部包含很多流程，

例如依赖安装、编译、测试、部署等。

#### 2.2.2 Stages

Stages：为我们上面提到一个Pipeline中的各个流程，Pipeline为一个总的构建任务，内部包含多个Stages，Stages执行任务需要遵循一些流程规则

* 所有的Stages都是从上到下顺序执行，当第一个Stages执行完成后，才开始下个Stages，以此类推；
* 当其中任何一个Stages执行失败后，由于是顺序执行后面的Stages将不会被执行，为了保证一致性，该Pipeline则为失败状；
* 当一个Pipeline中所有的Stages都成功执行完成后，该Pipeline状态才为成功。

#### 2.2.3 Jobs

我们可以理解Pipeline为一个总体的大任务，完成这个大任务，需要很多步骤，这些步骤就是Stages，每个Stage有可以有多个jobs来组成，同Stages一样，jobs的运行也需要遵循一些流程规则

* 在同一个Stage中可能含有多个Job，这些Jobs在执行过程中会并行执行；
* 一个Stage中的任何一个job执行失败，那么该Stage失败，同样的Pipeline状态也为失败；
* 只有当一个Stage中的所有job都执行成功时，该Stage才为成功。



了解了上面的基本概念，我们以图的形式再来回顾下各个概念：

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200216222506.png)

如上图，一个Pipeline为一次构建任务，其中包含多个步骤Stage，例如安装依赖/测试/部署等，例如在安装依赖一个Stage内又有多个Job，这些job并行运行，例如安装多个软件，这些Job如果有一个失败，那个这个Stage就为失败，Stage是从上到下顺序执行，如果其中有一个Stage状态失败，则本次构建任务也就是这个Pipeline就是失败的。

## 三 Gitlab CI/CD工作流程

在我们了解了Gitlab CI的相概念后，让我们来全局性的了解Gitlab CI/CD的工作流程，如下图所示：

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200216232000.png)

我们可以从图中看到Gitlab 的CI/CD工作流只要分为三个阶段

### 3.1 校验

在该步骤中Gitlab 主要完成CI功能，

* 开放人员提交代码到一个特定的分支，改分支具有Gitlab CI
* Gitlab CI完整构建在其中有代码扫描，性能测试，单元测试，容器扫描，依赖扫描等
* 在其中Gitlab的那个stage失败后，需要开发者去进行对应修复，直到该分支的pipeline都构建和测试正常

### 3.2 打包

* 之后可以通过应用进行预览应用，最终将成功的进行制品管理
* 通过：[Container Registry](https://docs.gitlab.com/ee/user/packages/container_registry/index.html)存储docker镜像
* 通过：[NPM Registry](https://docs.gitlab.com/ee/user/packages/npm_registry/index.html)存储NPM包
* 通过[Maven Repository](https://docs.gitlab.com/ee/user/packages/maven_repository/index.html)存储Maven工件
* 通过：[Conan Repository](https://docs.gitlab.com/ee/user/packages/conan_repository/index.html)存储Conan包

### 3.3 发布

* 持续部署，将通过测试和构建的分支merge 到主分支
* 通过手动确认审批部署将应用部署进入正式环境，进行应用发布
* 通过[GitLab Pages](https://docs.gitlab.com/ee/user/project/pages/index.html)部署静态网站
* 进行金丝雀发布，让一定比例的用户群新功能
* 通过[GitLab Releases](https://docs.gitlab.com/ee/user/project/releases/index.html)增加发新的记录

## 四 Gitlab CI中的关键字

### 4.1 关键字

在上面我们全局的了解Gitlab CI/CD的工作流程后，那们我们该怎样开始我们的CI之旅呢，在开始之前，我们需要对应gitlab-ci.yml中的核心关键字需要有非常清楚的认识，在后续实战中，完成具体的CI功能全部都是依赖于这些关键字,在此仅列出一些我们经常用的核心关键字，后续如果根据实战案例有自己自定义的需求，可以参考官方文档进行查阅：https://docs.gitlab.com/ce/ci/yaml/README.html

注意：在以下的核心关键字作为Gitlab CI内置的保留字段，不能够作为job的名称

| 关键字        | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| Stages        | 定义构建任务，可以被job所引用，可以是一系列job的集合         |
| Jobs          | 定义真正实现执行任务，每个jobs必须有一个唯一的名字           |
| variables     | 在CI文件中编写变了，并在job环境中起作用，`CI_COMMIT_REG_NAME`就是一个很好的例子，它的值表示用于构建项目的分支或tag名称，除了在`.gitlab-ci.yml`中设置变量外，还有可以通过GitLab的界面上设置私有变量。 |
| environment   | 用于定义job部署到特殊的环境中。如果指定了`environment`，并且没有该名称下的环境，则会自动创建新环境。 |
| scripts       | Runner执行的命令或脚本，该关键子为必选                       |
| tags          | 应用指定注册在该项目下的gitlab runner（同时Runner也要设置tags） |
| except        | 在一个job下，指定一个git分支不能执行此构建，如果`only`和`except`在一个job配置中同时存在，则以`only`为准 |
| only          | 在一个job下，指定一个git分支可以执行此构建                   |
| when          | 定义何时开始job。可以是`on_success`，`on_failure`，`always`或者`manual` |
| before_script | 重写一组在作业前执行的命令                                   |
| after_script  | 定义此作业完成部署的环境名称                                 |
| cache         | 定义一组文件列表，可在后续的stage使用。                      |
| image         | 执行stage使用的docker镜像                                    |

### 4.2 示例

上面我们了解了一些核心的关键字，下面让我们以一个前端项目的示例来更为直观的对关键字进行说明

```yaml
stages:        # 定义该pipeline执行的stage有三个，结构为列表
  - build      # 构建stage
  - test       # 测试stage
  - deploy     # 部署stage
  
variables:                                 # 定义变量，结构为字典
  BASE_DIR: "/data/frontendadmin/"         # 变量的key为BASE_DIR， value为具体的主路径

cache:									# 定义缓存
  paths:								# 定义缓存的目录，结果为列表
  - node_modules/				# 具体需要缓存的路径
  - dist/

before_script:																				# 在开始job之前执行的操作，一般用来初始化环境等
  - cp ${BASE_DIR}.env.example ${BASE_DIR}.env
  - yarn install
  - yarn build

build_job:										# 定义构建job的名称
  stage: build								# 具体饮用stages中的job，需要名字保持一致
  script:											# 在该job中执行的脚本
   - yarn build
  tags:												# 选择使用那个gitlab runner来执行此job
    - dev
  only:
    - develop									# 选定改项目git仓库中的develop分支进行执行改job
  when: always								# 无论什么条件都执行改job

test_job:										
  stage: test
  script:
   - yarn test
  tags:
    - dev
  only:
    - develop
  when: manual								# 指定需要手动点击才执行改job

deploy_job:
  stage: deploy
  only:
   - master										# 仅对master分支进行执行改job
  script:
   - pm2 delete api || true
   - pm2 start dist/index.js --name api
  tags:
    - dev
  when: always
```

通过上门这个`.gitlab-ci.yml`我们能够更加更加直观的认识到Gitlab CI中的一些关键字的作用和整个CI文件的大体结构，

在这个pipeline中，定义来三个stage

* 在全局定义了环境变量BASE_DIR，可以在后续的job中进行引用，如果定义在某一个job中则只能被改job引用，引用方式:${BASE_DIR}
* 定义了cache环境，用户为各job中传递一些持久化的缓存文件,例如在java项目中编译的jar/war包等。

* 利用before_script，在运行stage前对我们部署的环境进行了初始化

* build：不管什么情况下，也就是总是对项目的develop分支利用tag为dev的gitlab runner进行构建
* test：需要手动去点击执行job，同样对develop分支进利用tag为dev的gitlab runner执行测试
* deploy：总是对master分支，利用tag为dev的gitlab runner执行改部署job

上面的示例是为了我们了解各关键字而设立的一个前端项目，在后面章节的实战项目中，我们能够结合实际案例来更清楚的了解每一个步骤，同时我们可以对自己现有的项目的CI进行抽象化为各个stage，自己去编写`.gitlab-ci.yml`进行不断的测试来了解关键字即结构。

## 五 总结

通过该章节我们能够系统的学习到gitlab中的关键组建的含义/分类以及`.gitlab-ci.yml`的语法及关键字，其次对Gitlab CI/CD的工作流程有了直观的认识，通过实例更清楚直观的学习其中关键字的用户和CI文件的结构。

总结来说，想要使我们的项目能够集成Gitlab CI，只需要在项目中注册gitlab runner，其次在项目目录下根据自己业务编写`.gitlab-ci.yml`即可，至此我们的项目已经拥有了CI的能力，那么我们在什么样的场景下该如何根据自己的业务编写CI文件呢，后续我们将几个典型的实战项目，从零到一实战Gitlab CI，相信通过实战定会对你后续的项目有更深层次的理解和认识。