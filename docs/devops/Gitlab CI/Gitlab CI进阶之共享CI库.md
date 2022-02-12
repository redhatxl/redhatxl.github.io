# Gitlab CI进阶之共享CI库

# 一 背景

目前对于gitlab CI是在单独的项目下创建.gitlab-ci.yaml文件来定义部署过程，对于共同的一些步骤比如构建部署等，在每一个gitlab CI文件中编写，为了能够使代码在不同项目复用，将其存放在一个专门用于构建的gitlab CI仓库，其他项目想要使用该stage可以引用公共的CI文件，后续仅需要维护公共的gitlab CI库即可，但是需要公共CI库将一些特征数据提取出来，由 CI 确保代码风格一致，并执行单元测试和静态检查等。由于仓库数量众多，如何有效地组织和管理 CI 配置成了问题。经过长时间的探索和优化，我整理了一些经验，希望对你有所帮助。公共库需要较好的扩展性与兼容性。

# 二 基础语法解析

在共享仓库中将单个操作抽象为一个原子jobs，单独写在一个文件中，这样可以在模版中引用这些原子jobs，根据不同的变量，tags的runner，及branch可以任意组合成需要的模版。对此主要用到两个Gitlab CI中的关键字，include和extends。

include和extends是配合使用的，include为引用项目中的yaml文件，extends，为继承文件中的具体jobs

## 2.1 Include

### 2.1.1 功能

利用`include`关键字能够引用其他外部的yaml文件，这有助于将CI/CD配置分解为多个文件，并提高长配置文件的可读性。

* 可以将通用的一些操作，抽象为单个原子jobs，编写共享gitlab ci库，在模版中incloud这些原子操作jobs，来组合成模版。
* 在单个的项目中，可以include共享库中预定于好的模版，仅重写项目中的一些变量即可。

### 2.1.2 方式

include加载其他外部yaml文件，文件名称扩展必须为`.yaml`或`.yml`。

include支持加载方法包含以下四种：

| Method                                                       | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [`local`](https://docs.gitlab.com/ee/ci/yaml/#includelocal)  | Include a file from the local project repository.            |
| [`file`](https://docs.gitlab.com/ee/ci/yaml/#includefile)    | Include a file from a different project repository.          |
| [`remote`](https://docs.gitlab.com/ee/ci/yaml/#includeremote) | Include a file from a remote URL. Must be publicly accessible. |
| [`template`](https://docs.gitlab.com/ee/ci/yaml/#includetemplate) | Include templates which are provided by GitLab.              |

### 2.1.3 示例

* include:local:

include:本地包含与.gitlab来自同一存储库的文件-词yml. 它是使用相对于根目录（/）的完整路径引用的。

您只能在配置文件所在的同一分支上使用Git当前跟踪的文件。换句话说，当使用包括：本地，确保两者都是。gitlab-词yml本地文件在同一个分支上。

```yaml
include:
  - local: '/templates/.gitlab-ci-template.yml'
```

或者简短的方法：

```yaml
include: '.gitlab-ci-production.yml'
```

* include:file

要在同一个GitLab实例下包含来自另一个私有项目的文件，请使用包括：文件。使用相对于根目录（/）的完整路径引用此文件。例如：

```yaml
include:
  - project: 'my-group/my-project'
    file: '/templates/.gitlab-ci-template.yml'
```

也可以指定ref.

```yaml
include:
  - project: 'my-group/my-project'
    ref: master
    file: '/templates/.gitlab-ci-template.yml'

  - project: 'my-group/my-project'
    ref: v1.0.0
    file: '/templates/.gitlab-ci-template.yml'

  - project: 'my-group/my-project'
    ref: 787123b47f14b552955ca2786bc9542ae66fee5b # Git SHA
    file: '/templates/.gitlab-ci-template.yml'
```

* include:remote

Remote用于包含来自不同位置的文件，使用HTTP/HTTPS，通过使用完整的URL引用。远程文件必须通过简单的GET请求公开访问，因为远程URL中的身份验证架构不受支持。例如：

```yaml
include:
  - remote: 'https://gitlab.com/awesome-project/raw/master/.gitlab-ci-template.yml'
```

* include:template

利用template可以加载gitlab上已经预置的一些模版，

https://gitlab.com/gitlab-org/gitlab/tree/master/lib/gitlab/ci/templates

```yaml
# File sourced from GitLab's template collection
include:
  - template: Auto-DevOps.gitlab-ci.yml
```

也可以引用多个模版

```yaml
include:
  - template: Android-Fastlane.gitlab-ci.yml
  - template: Auto-DevOps.gitlab-ci.yml
```

## 2.2 Extends

### 2.2.1 功能

可以把一些公共属性或者方法（主要是Script）也进行统一管理。将其抽离在单独的jobs中，在具体的stages中进行继承。

extends定义使用extends的作业将从中继承的条目名。

它的使用相较于yaml anchors，更灵活和更易读。

### 2.2.2 示例

```yaml
.tests:
  script: rake test
  stage: test
  only:
    refs:
      - branches

rspec:
  extends: .tests
  script: rake rspec
  only:
    variables:
      - $RSPEC
```

在上述的示例中，rspec 的job内嵌了.tests模版的job，gitlab将要执行深度合并,结果为：

```yaml
rspec:
  script: rake rspec
  stage: test
  only:
    refs:
      - branches
    variables:
      - $RSPEC
```

# 三 实战

## 3.1 项目结构

```yaml

├── README.md
├── jobs                                # 基础jobs
│   ├── build.yaml                      # 构建jobs
│   ├── check.yaml                      # k8s 部署检测jobs
│   ├── deploy.yaml                     # k8s 部署jobs
│   ├── scan.yaml                       # k8s 代码扫描jobs
│   └── test.yaml                       # k8s dan单元测试jobs
└── templates                           # pipelines 模版
    ├── backend-k8s-private.yaml        # 私有化后端部署模版
    ├── backend-k8s-saas.yaml           # saas后端部署模版
    ├── frontend-k8s-private.yaml       # 私有化前端部署模版
    └── frontend-k8s-saas.yaml          # saas前端部署模版

```

* 将原子操作抽象成单个jobs
* 在templates中定义后端私有化几saas 的CI模版，在其中引用

## 3.2 CI jobs及模版

### 3.2.1 jobs

在jobs中定义原子操作，在此实例build

```yaml
.build:
  script:
    - echo -e "\033[5;35;40m Building Dockerfile-based application ${REGISTRY}/${NAMESPACE}/${APP_NAME}:${APP_TAG}... \033[0m"
    - docker build -t "${REGISTRY}/${NAMESPACE}/${APP_NAME}:${APP_TAG}" .
    - echo "${REGISTRY_PWD}" | docker login ${REGISTRY} -u "${REGISTRY_USER}" --password-stdin
    - docker push "${REGISTRY}/${NAMESPACE}/${APP_NAME}:${APP_TAG}"
    - docker rmi ${REGISTRY}/${NAMESPACE}/${APP_NAME}:${APP_TAG}
  retry:
    max: 2
    when:
      - always

.upload:
  script:
    - cd /builds/${PROJECT_GROUP}
    - tar -zcf /tmp/${PROJECT_NAME}.tgz ${PROJECT_NAME}
    - coscmd config -a${ACCESS_ID} -s${ACCESS_KEY} -b${BUCKET} -r${REGION}
    - coscmd upload /tmp/${PROJECT_NAME}.tgz ${OBJECT_PATH}/
  retry:
    max: 2
    when:
      - always
```

我们可以其中定义build和upload多个操作，在其中不定义stage/branch/tags等。

### 3.2.2 templates

在templates中定义不同场景的CI模版，在其中可以自己根据项目是私有化部署或是saas 来include不同的jobs文件，并继承其中具体jobs。 

在自定义模版中，主要包含四个部分：

* 导入jobs的yaml文件
* 定义全局就stage中变量
* 定义模版运行CI的stage
* 实现具体stage，管理具体stages，继承jobs。

在此义deploy来示例：

```yaml
# 1.导入作业模版
include:
  - project: 'devops/gitlabci-templates'
    ref: master
    file: 'jobs/test.yaml'
  - project: 'devops/gitlabci-templates'
    ref: master
    file: 'jobs/scan.yaml'
  - project: 'devops/gitlabci-templates'
    ref: master
    file: 'jobs/build.yaml'
  - project: 'devops/gitlabci-templates'
    ref: master
    file: 'jobs/deploy.yaml'
  - project: 'devops/gitlabci-templates'
    ref: master
    file: 'jobs/check.yaml'

# 2.定义全局及stage变量
variables:
  # 2.1 全局变量
  # 2.1.1 全局基本不用修改变量
  # 全局默认构建容器
  IMAGE: docker:latest
  # 镜像仓库
  REGISTRY: ccr.ccs.tencentyun.com
  # gitlab项目组
  PROJECT_GROUP: devops
  # 部署配置
  DEPLOY_DIR: deploy
  DEPLOY_NAME: deployment.yaml
  # 配置文件
  CONFIG_DIR: cfg
  CONFIG_KEY: config.yaml

  # 部署不同环境的目录
  DEV_ENV_DIR: dev
  TEST_ENV_DIR: test
  PROD_ENV_DIR: prod

  # 配置namespace，该名称空间在镜像仓库和k8s中名称空间保持一致
  DEV_NAMESPACE: anchnet-devops-dev
  TEST_NAMESPACE: anchnet-devops-test
  PROD_NAMESPACE: anchnet-devops-prod
  # 定义各阶段使用镜像
  SCAN_IMAGE: sonarsource/sonar-scanner-cli:latest
  TEST_IMAGE: python:3.6
  UPLOAD_IMAGE: ccr.ccs.tencentyun.com/anchnet-devops-common-test/python-coscmd:latest
  DEPLOY_IMAGE: kubesphere/kubectl:v1.0.0
  # 定义上传到cos配置
  OBJECT_PATH: /smartant/saas
  BUCKET: go2tencent-1253329830
  REGION: ap-shanghai

  # 2.1.2 全局需要根据项目自定义变量
  # 主要用于对k8s资源操作
  APP_NAME: smartant-backend
  # 项目名称与代码仓库名称保持一致，用于对目录操作
  PROJECT_NAME: smartant_backend


  # 2.2 各个stage变量
  # 2.2.1 代码扫描测试，test-scan变量
  REQUIREMENTS: requirements/requirements.txt
  TEST_DIR: test
  TEST_PARAM: --include=../application.py,../logs.py,../libs/*.py,../views/*.py  --omit="test_*.py" runtests.py
  # 2.2.2 构建和上传源码到cos，build-upload变量


  # 2.2.3 部署到k8s，deploy变量
  DEV_CONFIG_FILE: config_dev.yaml
  TEST_CONFIG_FILE: config_test.yaml
  PROD_CONFIG_FILE: config_prod.yaml
  # 部署在k8s中k8s的configmap名称，一般为APP_NAME-cm
  CM_NAME: smartant-backend-cm

  # 2.2.4 检查k8s deploy服务，check变量


before_script:
  - export APP_TAG="${CI_COMMIT_TAG:-${CI_COMMIT_SHA::8}}"

# 3.配置运行stage
stages:
  - test-scan
  - build-upload
  - deploy
  - check


# 4.作业配置

image: ${IMAGE}


# --------------test and scan stage----------------
# dev环境代码扫描
scan-dev:
  image: ${SCAN_IMAGE}
  tags:
    - devops-dev-runner
  stage: test-scan
  extends: .scan
  only:
    - dev

# dev环境单元测试
test-dev:
  image: ${TEST_IMAGE}
  tags:
    - devops-dev-runner
  stage: test-scan
  extends: .test
  only:
    - dev


# --------------build upload stage---------------
# dev环境构建镜像
build-dev:
  variables:
    NAMESPACE: ${DEV_NAMESPACE}
  tags:
    - devops-dev-runner
  stage: build-upload
  extends: .build
  only:
    - dev


# test环境构建镜像
build-test:
  variables:
    NAMESPACE: ${DEV_NAMESPACE}
  tags:
    - devops-test-runner
  stage: build-upload
  extends: .build
  only:
    - test

# 正式环境构建镜像
build-prod:
  variables:
    NAMESPACE: ${PROD_NAMESPACE}
  tags:
    - devops-prod-runner
  stage: build-upload
  extends: .build
  only:
    - master

# 正式环境打包代码上传至cos
upload-prod:
  image: ${UPLOAD_IMAGE}
  variables:
    NAMESPACE: ${PROD_NAMESPACE}
  tags:
    - devops-prod-runner
  stage: build-upload
  extends: .upload
  only:
    - master

# --------------deploy stage---------------
# dev环境发布pages
pages:
  tags:
    - devops-dev-runner
  stage: deploy
  dependencies:
    - test-dev
  extends: .pages
  only:
    - dev

# dev 环境部署
deploy-dev:
  image: ${DEPLOY_IMAGE}
  variables:
    NAMESPACE: ${DEV_NAMESPACE}
    DEPLOY_ENV: ${DEV_ENV_DIR}
    CONFIG_FILE: ${DEV_CONFIG_FILE}
  tags:
    - devops-dev-runner
  stage: deploy
  extends: .deploy
  only:
    - dev

# test环境部署
deploy-test:
  image: ${DEPLOY_IMAGE}
  variables:
    NAMESPACE: ${TEST_NAMESPACE}
    DEPLOY_ENV: ${DEV_ENV_DIR}
    CONFIG_FILE: ${DEV_CONFIG_FILE}
  tags:
    - devops-test-runner
  stage: deploy
  extends: .deploy
  only:
    - test

# --------------check stage---------------
# dev环境k8s测试
check-dev:
  image: ${DEPLOY_IMAGE}
  variables:
    NAMESPACE: ${DEV_NAMESPACE}
  tags:
    - devops-dev-runner
  stage: check
  extends: .check
  only:
    - dev

# test环境k8s测试
check-test:
  image: ${DEPLOY_IMAGE}
  variables:
    NAMESPACE: ${TEST_NAMESPACE}
  tags:
    - devops-test-runner
  stage: check
  extends: .check
  only:
    - test

```

在该模版中利用stage中的image/tags/runner/branch进行灵活组合，从而实现适应不同场景的CI。

至此共享CI模版库就已经创建完成，需要在具体的项目中进行应用。

## 3.3 项目中引用

在具体的项目中引用非常简单，只需要在代码仓库中编写`.gitlab-ci.yaml`即可实现CI继承，在此我们示例一个python的后端saas项目，继承backend的saas模版。

```yaml
include:
  - project: 'devops/gitlabci-templates'
    ref: master
    file: 'templates/backend-k8s-saas.yaml'

variables:
  APP_NAME: smartant-api-linux
  # 项目名称与代码仓库名称保持一致，用于对目录操作
  PROJECT_NAME: smartant_api_linux

  # 2.2 各个stage变量
  # 2.2.1 代码扫描测试，test-scan变量
  REQUIREMENTS: requirements/requirements.txt
  TEST_DIR: test
  TEST_PARAM: --include=../application.py,../logs.py,../libs/*.py,../views/*.py  --omit="test_*.py" runtests.py
  # 2.2.2 构建和上传源码到cos，build-upload变量

  # 2.2.3 部署到k8s，deploy变量
  DEV_CONFIG_FILE: config_dev.yaml
  TEST_CONFIG_FILE: config_test.yaml
  PROD_CONFIG_FILE: config_prod.yaml
  # 部署在k8s中k8s的configmap名称，一般为APP_NAME-cm
  CM_NAME: smartant-api-linux-cm
```

可以看到在具体的项目中集成gitlab CI引用模版库非常的简单，只用引用文件，并根据项目来定义变量即可完成集成。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200707171423.png)

# 四 注意事项

## 4.1 引用注意

* 引用的时候如果为同一个仓库可以使用include:local进行同一个仓库文件本地引用
* 对于在同一个gitlab服务器上的项目，可以使用project来引用，注意引用末尾没有.git

## 4.2 共享库注意事项

* jobs中的原子操作不用定义stages，和branch，及运行的gitlab runner的tags
* 将公共变量提出来，将根据项目定义的变量也提出来，提高共享库的可扩展性及灵活性
* 在具体项目中引用模版，重新定义变量，将覆盖全局变量

# 五 反思

为了提升CI复用性和扩展性及规范CI流程，gitlab CI共享库非常好的解决了此问题，但是要求编写的具体jobs需要能无状态化，需要具备很高的扩展性和维护行，对于前期的规划和编写jobs都提出了很高的要求。

# 六 参考链接

* https://docs.gitlab.com/ee/ci/yaml/#include
* https://docs.gitlab.com/ee/ci/yaml/includes.html
* https://www.jianshu.com/p/cc07db6de12d