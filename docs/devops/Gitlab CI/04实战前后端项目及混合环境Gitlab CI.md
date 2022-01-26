# 4.实战前后端项目及混合环境Gitlab CI

## 一 背景

我们上一节以一个简单Python项目示例为我们展示了Gitlab CI的魅力，在其中我们已经讲解了Gitlab runners的安装配置以及相关查看方式。本章节我们来实战Gitlab CI在前端及编写打包形项目中的应用，以及示例第三方工具的集成。

## 二 工具集成

### 2.1 Gitlab CI集成SonarQube

我们知道，对于项目代码质量管理，在目前的微服务/模块化/快迭代敏捷开发中如果仅依赖IDE简单检查和人为的codereview对于大量代码很不适合，不仅仅依靠开发人员的编码规范编码，更需要一些工具来帮助我们提前预防和强制检测规范。在我们代码开放中有一项非常重要的事情就是代码漏洞及安全扫描，我们可以将SonarQube代码扫描即成在我们的Gitlab CI中，每次任务跑完，可以自己去检测扫描出来的漏洞进行修复，在此我们见来实践下Gitlab CI集成SonarQube，

#### 2.1.1 SonarQube的介绍

##### 2.1.1.1 简介

Sonarqube 是一款代码分析检测工具，将其与我们的CI/CD结合，例如集成到gitlab ci/cd或jenkins中实现部署自动代码检查，及时发现并处理bug，最大限度的将bug和不规范扼杀在编码阶段，其内部集成很多分析工具，比如pmd-cpd、checkstyle、findbugs、从七个方面帮我们来源码质量管理。

##### 2.1.1.2 特点

- 检查代码是否遵循编程标准：如命名规范，编写的规范等。
- 检查设计存在的潜在缺陷：SonarQube 通过插件 Findbugs、Checkstyle 等工具检测代码存在的缺陷。
- 检测代码的重复代码量：SonarQube 可以展示项目中存在大量复制粘贴的代码。
- 检测代码中注释的程度：源码注释过多或者太少都不好，影响程序的可读可理解性。
- 检测代码中包、类之间的关系：分析类之间的关系是否合理，复杂度情况。

##### 2.1.1.3 组件 

- SonarQube Server：sonarqube服务端，接受客户端扫描报告。
- SonarQube Database：ES/及数据库引擎oracle，postgresql，mssql。
- SonarQube Plugins：可以后期在sonarqube服务端安装插件。
- SonarQube Scanner：安装在客户端扫描工具。

在此我们的重点为Gitlab CI因此就不过多的介绍关于SonarQube的相关知识，我们知道Gitlab CI集成软件只需要在common-server上安装工具或者客户端接口，因此在此处我们为们在gitlab-common-runner上面安装SonarQube Scanner即可。

##### 2.1.1.4 流程

我们知道了如果将SonarQube Scanner集成到Gitlab CI中，让我们以下图更直观的形式来分析其流程。

![img](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200223000957.png)

开发人员把代码push到SCM（如gitlab），由于项目存在`.gitlab-ci.yml`文件触发Gilab CI，在其中的stage里面定义代码扫描的job，在common-runner服务器上调用已经安装好的SonarQube Scanner来扫描项目代码，扫描完成后将生成的xml报告push到SonarQube Server上，此刻我们就可以在SonnarQube Server查看报告。

#### 2.1.2 配置部署

在此SonarQube服务端的安装配置我们就不再此过多的篇幅展开，主要集中在与Gitlab CI的集成这边

##### 2.1.2.1 SonarQube项目配置

Create new project->Provide a token->

![img](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200223003619.png)

##### 2.1.2.2 记录脚本命令

在此我们需要根据具体项目主要的编写语言及操作系统来选择执行脚本

![img](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200223003601.png)

##### 2.1.2.3 与Gitlab CI集成

```
 stages:
   - sonarqube_scan						# 指定扫描stage
 
 job sonarqube_scan_job:
   stage: sonarqube_scan
   # 注意，此用户为gitlab-runner执行，指定/.为此项目目录
   script:
     - sonar-scanner -Dsonar.projectKey=go2cloud_api_test -Dsonar.sources=/. -Dsonar.host.url=http://43.xxx.xxx.xxx:9110 -Dsonar.login=a393276xxxxxxxxxxxxxxxxxxx03004a714
   tags:
     - common-runner					# 用来运行扫描的gitlab runner，该runner上需要安装sonar-scanner
   only:
     - go2cloud-platform-test			# 仅在项目的该branch进行扫描
   when: always
```

如上就完成了我们的sonarqube的集成，只需要在gitlab-runner上安装扫描客户端，在我们的CI的一个stage中进行引用即可。

## 三 前后端项目

在我们实际的项目中，一个具体的项目我们一遍都是前后端分离，此时该如何进行一个项目的统一发布，gitlab CI在前端应用中又该如何使用，让我们一块实践前后端业务的发布。

### 3.1 前端项目

#### 3.1.1 项目分析

前端项目目前主流的例如vue，我们以此项目进行举例

* 首选需要在项目内自定义打包的目录及打包的相对路径

```js
const path = require('path')

const CaseSensitivePathsPlugin = require('case-sensitive-paths-webpack-plugin')

function resolve(dir) {
  return path.join(__dirname, dir)
}

module.exports = {
  configureWebpack: {
    resolve: {
      extensions: ['.js', '.vue', '.json', '.scss', '/index.vue', '/Index.vue']
    },
    plugins: [
      new CaseSensitivePathsPlugin()
    ]
  },
```

* 在common-runner服务器需要部署安装部署node，利用npm来打包前端项目
* 在目标服务器上需要Web服务器，例如nginx，需要事先部署好，并定义好web目录，将我们编译打包好的静态文件发布上去，我们可以使用为web服务器的网站数据目录做一个链接，之后利用rsync将每次编译好的文件同步到目标目录，实现静态文件的发布。

#### 3.1.2 架构图

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200223153033.png)

此处为们例如共用这个comm-runner来做代码扫描并利用在其上安装的node环境进行前端项目打包，最终将打包好的静态文件rsync到目标服务器nginx的web目录下。

当然也可以直接将在目标服务器上安装node，在目标服务器打包然后rsync到nginx的web目录下。

#### 3.1.3 集成CI

在此这个前端项目中刚好我们也顺便演示下SonarQube的集成，将其作为一个stage，让我们直接来开始看`.gitlab-ci.yml`文件

```yaml
stages:
  - sonarqube_scan		  # 代码扫描
  - deploy_src					# 打包部署
  - check_server				# 服务检测

variables:
  RUNNER_HOME: "/home/gitlab-runner/builds/1sWu8AdN/0/devops"
  BASE_DIR: "/smartms_frontendadmin/"

job sonarqube_scan_job:
  stage: sonarqube_scan
  script:
    - sonar-scanner -Dsonar.projectKey=smartms_frontendadmin_test -Dsonar.sources=. -Dsonar.host.url=http://43x.xx.xx.xx -Dsonar.login=a3932xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx714			# 代码扫描
  tags:
    - smartms-frontend-test
  only:
    - develop
  when: always

job deploy_src_job:
  stage: deploy_src
  cache:
    paths:
    - node_modules/
  script:
    - cnpm install											# 安装依赖
    - cnpm install vue-cli
    - cnpm run build 										# 构建
    - rsync -az --delete ${RUNNER_HOME}${BASE_DIR}dist/ root@172.15.100.2:/data/dist/			# 同步文件
  tags:
    - smartms-frontend-test
  only:
    - develop
  when: always

job check_server_job:
  stage: check_server
  script:
    - ssh -o stricthostkeychecking=no root@172.16.100.2 ps -ef|grep "nginx" 				# 检测进程
  tags:
    - smartms-frontend-test
  only:
    - develop
  when: always
```

在此前端项目中我们进行了三个大的步骤，

* 代码扫描：利用集成的sonar-scanner来对代码进行扫描，将报告发送至服务端。
* 打包部署：在此过程中，我们定义了cache目录为`node_modules`为缓存目录，之后利用cnpm对我们的项目进行打包，最后利用rsync将生成好的静态文件从那个dist/ 目录同步到目标的nginx服务器的web目录下进行发布。
* 服务检测：我们在common-runner服务器上检测目标服务器nginx是否运行正常。

当然在其中可以针对最近的业务进行更详细的配置，例如单元测试，通知等都可以进行自定义。我们在发布完成后可以通过钉钉机器人或者企业微信消息通知相关人员，如果有监控工具，同样可以将构建的相关数据指标发送至监控Server端口，来进行告警通知。

#### 3.1.3 测试

* 查看pipeline运行状态

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200223154155.png)

* 查看打包部署的stage

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200223154246.png)

* 最终为们可以浏览器访问我们的nginx进行页面测试，最大程度的隐藏了部署的细节，不用关系每次部署发布。

### 3.2 后端项目

#### 3.2.1 项目分析

后项目例如之前为们发布的Python是以源码的方式，在此为们以Java项目为例，需要打包，最终发布到目标服务器。

- 需要在中转服务器进行打包工具的安装，在此我们以gradlew 为例,需要提前在common-runner上面安装部署该工具。
- 在目标服务器需要如果为war包，需要提交安装部署tomcat等web服务，如果为jar包则可以编写脚本或利用supervisor来管理进程起停。
- 后端项目需要能够正连接数据库，我们目标服务器需要能够正常访问数据库。

#### 3.2.2 架构图

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200223160844.png)

在此处我们共用这个comm-runner来做发布，需要在common-runner服务器上部署java项目的打包工具，最终将生成的制品rsync到目标服务器上，war包可以发布到tomcat的webapp目录下，jar包可以实先指定数据目录，最终重启tomcat或利用supervisor来管理进程。

当然也可以直接将在目标服务器上安装打包工具，在目标服务器之间进行发布。

#### 3.2.3 集成CI

```yaml
variables:
  APP_NAME: cmserver				# 定义app名称

build:dev:
  stage: build-src					# 部署的dev环境
  script:
    - ./gradlew build -x test
    - scp ./build/libs/$APP_NAME.jar ubuntu@172.16.100.3:/tmp
    - ssh ubuntu@172.16.100.3 /etc/init.d/$APP_NAME stop
    - ssh ubuntu@172.16.100.3 cp -a /tmp/$APP_NAME.jar /opt/$CI_PROJECT_PATH/
    - ssh ubuntu@172.16.100.3 /etc/init.d/$APP_NAME start
  tags:
    - common-runner
  only:
    - dev										# 根据此项目的dev分支进行发布
  when: manual							# 需要人手工确认
 
build:test:
  stage: build-src					# 部署的dev环境
  script:
    - ./gradlew build -x test
    - scp ./build/libs/$APP_NAME.jar ubuntu@192.168.201.118:/tmp
    - ssh ubuntu@172.16.100.34 /etc/init.d/$APP_NAME stop
    - ssh ubuntu@172.16.100.34 cp -a /tmp/$APP_NAME.jar /opt/$CI_PROJECT_PATH/
    - ssh ubuntu@172.16.100.34 /etc/init.d/$APP_NAME start
  tags:
    - common-runner
  only:
    - test									# 根据此项目的dev分支进行发布

    
publish-api-docs:
  stage: deploy-doc					# 发布文档
  script:
    - ./gradlew asciidoctor
    - scp build/asciidoc/html5/*.html root@172.16.100.7:/var/www/html/docs/$CI_PROJECT_PATH
  tags:
    - common-runner					
  only:
    - dev										# dev分支进行文档发布

```

在通过上面部署内容，我们能够看到该build分为了两个环境，对于多环境，我们指定dev分支自动发布到dev环境中，对于测试环境需要手工确认发布到test环境，同时对于dev环境自动发布文档。

在其中将发布的一些了操作写在了一个stage中，此种方法有利有弊

* 优点：如果发布多环境，由于在集成Gitlab CI只能在`.gitlab-ci.yml`一个文件中编写，将stage写的太多不利于多环境的发布管理
* 缺点：将所有job操作写在一个stage里面没有更为混淆，不便于相关人员查看job。

对此我们可以折中，对于多环境尽可能减少stage，这样利于多环境的管理，如果为单一环境可以将stage拆解开来。

#### 3.2.4 测试

* 查看job运行

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200223162911.png)

* 查看服务

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200223162858.png)

## 四 混合环境CI

### 4.1 项目背景

当不同项目组进行测试环境集成，目前遇到了dev环境在一台单独的云服务器，但是test环境在k8s中，利用gitlab ci实现持续集成，简单快速高效上线，这种情况下，我们的Gitlab CI该如何实现呢。

### 4.2 上线流程

首先提交自己的代码merge到dev环境后dev的gitlab ci pipeline自动构建部署到测试环境，构建成功后，先在测试环境进行功能测试及bug fix，待一个大版本功能完成后将dev merge到master分支，master执行master分支的gitlab ci pipeline，进行正式环境k8s的构建部署。

### 4.3 环境差异

目前在项目下注册了两个gitlab runner来分别执行不通分支的ci任务

* 两个不同的runner负责执行不同环境的ci job，目前发布测试环境利用的是与k8s的node节点网络可以互通的云服务器，后续可以改进为运行在K8s中的POD形式，可以来并发执行，且可以保证环境一致性。 注意⚠️：runner注册为gitlab-runner用户，需要切换到此用户进行k8s正式环境访问。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200223164158.png)


* 环境变量注入，为了避免敏感信息暴露在代码配置文件中，将harbor仓库登录信息及kubeconf等敏感信息利用gitlab ci 的环境管理来进行注入

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200223164549.png)

### 4.4 集成CI

```yaml
stages:
  - deploy_src_dev						# 部署源码到dev环境
  - install_dependency_dev		# 安装依赖
  - restart_server_dev				# 重启服务
  - deploy_static_src_dev			# 部署静态文件
  - check_server_dev					# 检测服务
  - deploy_src_test						# 部署源码到test环境
  - build_image_test					# build docker镜像
  - deploy_k8s_test						# 部署进k8s
  - check_k8s_test						# 检测服务


variables:
  DEV_RUNNER_HOME: "/home/gitlab-runner/builds/ZxxxxxxxC/0/devops"
  PROD_RUNNER_HOME: "/home/gitlab-runner/builds/FKxxxxxF/0/devops"
  BASE_DIR: "/sxxxxxxxps/"
  APP_NAME: "smartcs_ops"
  PROJECET_NAME: "devops"


before_script:
  - export APP_TAG="${CI_COMMIT_TAG:-${CI_COMMIT_SHA::8}}"					# 定义Docker 的tag

job deploy_src_test_job:
  stage: deploy_src_dev
  script:
    - sudo rsync -az --delete ${DEV_RUNNER_HOME}${BASE_DIR}/ /opt${BASE_DIR}			# 同步文件
  tags:
    - smartops-opsaudit
  only:
    - dev
  when: always


job install_dependency_test_job:
  stage: install_dependency_dev
  script:
    - sudo /opt/py3/bin/python -m pip install -r /opt${BASE_DIR}requirements/requirements.txt			# 安装依赖
    - sudo \cp -rf /opt/config/config.yml /opt${BASE_DIR}/cfg/			# 拷贝配置文件
    - sudo mkdir -p /opt${BASE_DIR}/tmp
  tags:
    - smartops-opsaudit
  only:
    - dev
  when: always


job restart_server_test_job:
  stage: restart_server_dev
  script:
    - sudo /opt/py3/bin/supervisorctl restart all				# 利用supervisor管理dev环境进程
  tags:
    - smartops-opsaudit
  only:
    - dev
  when: always


job deploy_static_src_test_job:
  stage: deploy_static_src_dev
  script:
    - sudo mkdir /opt/smartcs-ops/data/ops-audit
    - sudo \cp -rf /opt/smartcs-ops/data/static/ /opt/smartcs-ops/data/ops-audit			# 同步文件
  tags:
    - smartops-opsaudit
  only:
    - dev
  when: always


job check_server_test_job:
  stage: check_server_dev
  script:
    - sudo /opt/py3/bin/supervisorctl status				# 利用supervisor管理进程
  tags:
    - smartops-opsaudit
  only:
    - dev
  when: always


job deploy_src_test_job:
  stage: deploy_src_test
  script:
    - sudo rsync -az --delete ${PROD_RUNNER_HOME}${BASE_DIR}/ /opt${BASE_DIR}			# 同步静态文件
    - sudo \cp -rf /opt${BASE_DIR}apps/static /opt${BASE_DIR}data
    - sudo mkdir /opt${BASE_DIR}data/ops-audit
    - sudo \cp -rf /opt${BASE_DIR}data/static/ /opt${BASE_DIR}data/ops-audit
  tags:
    - smartcs-ops-prod
  only:
    - master
  when: always


job build_image_test_job:
  stage: build_image_test
  script:
    - sudo chmod +x entrypoint.sh run_server.py
    - sudo docker build -t ${HARBOR_REGISTRY}/${PROJECET_NAME}/${APP_NAME}:${APP_TAG} .			# 制作镜像
    - sudo echo "${HARBOR_PASSWORD}" | docker login ${HARBOR_REGISTRY} -u "${HARBOR_USERNAME}" --password-stdin	# 登录dockerhub
    - sudo docker push ${HARBOR_REGISTRY}/${PROJECET_NAME}/${APP_NAME}:${APP_TAG}						# 推送镜像
  tags:
    - smartcs-ops-prod
  only:
    - master
  when: always


job deploy_k8s_test_job:
  stage: deploy_k8s_test
  script:
    - '[ -n "${KUBE_CONFIG_CONTENT}" ] && mkdir -p "${HOME}/.kube" && echo "${KUBE_CONFIG_CONTENT}" > "${HOME}/.kube/config"'						# 在此为将注入的kubeconfig文件生成文件
    - sudo grep -E "^[[:space:]]+image:" deploy-prod/*-deployment.yml |awk '{print $2}' |head -1			# 获取老的镜像
    - sudo sed -i "s@$(grep -E "^[[:space:]]+image:" deploy-prod/*-deployment.yml |awk '{print $2}' |head -1)@${HARBOR_REGISTRY}/${PROJECET_NAME}/${APP_NAME}:${APP_TAG}@g" deploy-prod/*-deployment.yml			# 替换为新镜像
    - kubectl apply -f deploy-prod/.					# 部署进k8s
  tags:
    - smartcs-ops-prod
  only:
    - master
  when: always


job check_backend_test_job:
  stage: check_k8s_test
  script:
    - kubectl rollout status -w --timeout=120s deployment/smartcs-ops-backend			# 检测rollout是否完成
  tags:
    - smartcs-ops-prod
  only:
    - master
  when: always


job check_celery_test_job:
  stage: check_k8s_test
  script:
    - kubectl rollout status -w --timeout=120s deployment/smartcs-ops-celery
  tags:
    - smartcs-ops-prod
  only:
    - master
  when: always


job check_beat_test_job:
  stage: check_k8s_test
  script:
    - kubectl rollout status -w --timeout=120s deployment/smartcs-ops-beat
  tags:
    - smartcs-ops-prod
  only:
    - master
  when: always
```

* deploy_src_prod 此步骤为拉取源码到prod服务器上，进行nginx静态文件的更新

* build_image_prod 此步骤为构建镜像，并推送镜像至harbor

* deploy_k8s_prod 此步骤为替换代码托管中的镜像tag，之后部署进k8s环境中

* check_k8s_prod 设立失败检测机制，通过rollout status -w在部署期间实时检测容器是否更新完成，如果容器有异常未启动，则状态为异常不会更新容器中的镜像，并行检测三个deployment的状态。

### 4.5 测试

* 查看pipeline

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200223165435.png)

* 查看正式环境部署pipeline

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200223165535.png)

至此我们就完成了混合环境的CI任务。

## 五 注意事项

* 在多环境配置下我们应该根据自身环境进行取舍，是拆分多个stage还是共用一个stage

* 此gitlab ci文件适用于利用gitlab ci完成CI的情况下，可以在一个.gitlab-ci.yml中编写多个环境的job。

* runner可以后期改造成docker形式，来并发执行多个job且保证环境一致性。

* 镜像的tag在此项目中定义为commit id，方便排查

* 将k8s的yaml文件托管在项目代码的目录结构下，每次仅替换image的tag，然后幂等性的apply。

## 六 反思

由于我们改小节为实战项目，其中包含了一些工具在此不便展开详解，这些如果大家有在工作中没有用的，可以去自己尝试学习使用，在此主要为大家提供一个思路，具体的细节需要自己在实际工作中亲自实践才能更加直观的认识。每个团队对应自己的业务都有特殊性，根据自身业务出发，找到最合适的方便就是最佳实践。





