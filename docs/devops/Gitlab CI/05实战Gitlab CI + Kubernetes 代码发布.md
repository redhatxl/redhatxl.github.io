# 5.实战Kubernetes Gitlab CI

## 一 背景

在目前微服务大行其道的背景下，Gitlab CI集成kubernetes已经是不可或缺的基本操作，我们前几节系统的实战了前后端项目以及物理/K8s混合环境部署，这节课我们来学习Gitlab CI如何将应用发布进K8s，我们都知道在之前的将gitlab-runner部署在服务器上面是存在一定的风险，如果运行pipeline的服务器宕机，发布任务就没办法继续了，更可怕的时候如果common-runner发送故障，多个发布任务就都有问题，在微服务架构中，不可变的基础设施，容器的自包含环境使得我们发布变得更加简单快捷，不用在考虑担心runner的环境如何根据不同的项目区分，且动态的Job触发，我们会随时拉起一个Pod来运行我们的Job，运行完成后又进行销毁，这样不仅能实现动态运行节省资源，而且可以不用考虑多项目多任务并发构建的问题，这节课就让我们来尽情享受K8s+Gitlab CI为我们带来畅快淋漓的发布体验。

## 二 架构解析

本文以构建一个Java软件项目并将其部署到阿里云容器服务Kubernetes集群中为例，说明如何使用GitLab CI在阿里云Kubernetes服务上运行GitLab Runner、配置Kubernetes类型的executor，并执行Pipeline。

### 2.1 Gitlab CI流程图

​	

### 2.2 流程详解

如上图为一个简单的Gitlab CI部署进K8s流程图，同之前我们讲到的CI集成一致，只需要项目中存在.gitlab-ci.yml文件即可，与之前的差异为在集成kubernetes的时候，我们讲我们的gitlab-runner运行在我们K8s内，其为一个POD形式运行，其控制着后续的pipeline中各stage的执行，我们可以看到，当一个pipeline有多个stage，每个stage都是有一个单独的Pod去执行，这个Pod使用的镜像在我们CI的.gitlab-ci.yml 的每个stage的image中定义，如所示为一个部署Java项目的流程

* 开发者或项目维护者merge request到特定分支，改分支存在CI文件，开始进行CI
* CI任务由已经注册在gitlab-server运行在K8s集群内的giitlab-runner进行下发
  * 第一个为package，对java项目利用maven镜像进行打包，生成war包制品到缓存目录中；
  * 利用docker镜像对根据项目中的Dockerfile对缓存中的制品进行镜像构建，构建完成后登录镜像仓库进行镜像推送；
  * 在此我们利用将部署文件托管在项目内，也体现了gitops的思想，将之前构建推送成功的镜像地址信息在deployment.yaml文件进行修改，之后apply进k8s中，java项目构建的镜像就运行在K8s集群内了，完成整个的发布。

在流程中有一些注意事项：

* 我们可以在stage中添加自己业务需求的内容，例如单元测试，代码扫描等。
* 在部署文件中，我们可以将整个项目制作成helm的chart，替换其中的镜像，利用helm来进行整个应用的部署。
* 应用在部署中单个stage利用的是不同的image，在各个stage中传递已经生成的制品例如war/jar包，需要使用到外部存储来缓存制品。

## 三  优点

通过上面的Gitlab CI流程我们能够看到将gitlab runner运行在K8s集群中，每个Job启动单独的POD运行操作，此种方式完全利用起了K8s的一些优点

* 高可用：当某个节点出现故障时，Kubernetes 会自动创建一个新的 GitLab-Runner 容器，并挂载同样的 Runner 配置，使服务达到高可用。
* 弹性伸缩：触发式任务，合理使用资源，每次运行脚本任务时，Gitlab-Runner 会自动创建一个或多个新的临时 Runner来运行Job。
* 资源最大化利用：动态创建Pod运行Job，资源自动释放，而且 Kubernetes 会根据每个节点资源的使用情况，动态分配临时 Runner 到空闲的节点上创建，降低出现因某节点资源利用率高，还排队等待在该节点的情况。
* 扩展性好：当 Kubernetes 集群的资源严重不足而导致临时 Runner 排队等待时，可以很容易的添加一个 Kubernetes Node 到集群中，从而实现横向扩展。

如果您的业务目前运行环境为K8s，那么Gitlab CI完全契合您的业务场景，您只需要自定义gitlab-ci.yml中的各个自己需求的stage即可，配合gitops将配置也托管在项目内，跟随项目一块维护管理，实现端到端的CI工作流，使得运维工作也可通过git追溯，提高工作效能，敏捷开发上线部署。

## 四 实战

我们在上面了解来Gitlab CI与Kubernetes的集成及其优点，下面就让我们通过实战来更具体的了解其流程。

### 4.1 环境准备

* 需要有Gitlab 服务器，可以是部署在物理服务器上，当然也可以部署在K8s集群内部。

* 我需要准备好K8s集群，可以为公有云的容器编排引擎，例如阿里的ACK，腾讯的TKE，华为的CCE等都适用这些方式。

* 由于gitlab-runner安装较为复杂，我们在示例中使用helm来进行安装，helm版本为v2.14.3，如果有能力可以自己编写资源清单部署。

#### 4.1.1 记录注册信息

登录Gitlab 服务器记录gitlab 的 url 和注册令牌，在我们部署进K8s的gitlab-runnner的配置中需要填写该信息，运行在K8s中的Pod就利用此此信息在gitlab服务器进行注册。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200227201303.png)

#### 4.1.2 获取gitlab-**runner**

由于单独部署gitlab-runner进K8s中，自己去写资源清单文件难度较大，而且容易出错，我们在此利用官方的chart镜像通过helm来进行部署，仅修改其中我们关系的字段即可，首先在登录K8s集群进行gitlab-runner的helm repo的添加，之后将chart下载到本地，解压文件并修改其中的values.yml文件。

* 添加repo获取charts

```shelll
[root@master common-service]# helm repo add gitlab https://charts.gitlab.io
"gitlab" has been added to your repositories
[root@master common-service]# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "aliyun" chart repository
...Successfully got an update from the "apphub" chart repository
...Successfully got an update from the "gitlab" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete.
[root@master common-service]# helm search gitlab-runner
NAME                    CHART VERSION   APP VERSION     DESCRIPTION  
gitlab/gitlab-runner    0.14.0          12.8.0          GitLab Runner
```

我看已经看到了 gitlab-runner的chart，其是 helm 中描述相关的一组 Kubernetes 资源的文件集合，里面包含了一个 value.yaml 配置文件和一系列模板（deployment.yaml、svc.yaml 等）。当然如果我们能力够，可以自己去编写这些资源清单文件。

在此有想去的伙伴可以查看之前K8s学习笔记，希望能对读者学习K8s有帮助：[awesome-kubernetes-notes](https://github.com/overnote/awesome-kubernetes-notes)

* 创建角色并绑定权限

gitlab-runner在运行的时候需要访问我们K8s的api，我们在此为期创建ServiceAccount并为其进行RBAC授权，首先创建一个gitlab-runners的名称空间，并创建对用的Role，将ServiceAccount于Role进行绑定。

```shell
[root@master gitlab-runner]# cat > rbac-runner-config.yaml <<EOF
apiVersion: v1
kind: ServiceAccount										# 在gitlab-runners名称空间下创建名为gitlab的serviceaccount
metadata:
  name: gitlab
  namespace: gitlab-runners
---
kind: Role																# 创建gitlab角色，
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: gitlab-runners
  name: gitlab
rules:
- apiGroups: [""] #"" indicates the core API group
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["apps"]								
  resources: ["deployments"]
  verbs: ["*"]
- apiGroups: ["extensions"]
  resources: ["deployments"]
  verbs: ["*"]

---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: gitlab																		# 将sa与角色进行绑定
  namespace: gitlab-runners
subjects:
- kind: ServiceAccount
  name: gitlab # Name is case sensitive
  apiGroup: ""
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: gitlab # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
  
EOF
# 创建资源清单
[root@master gitlab-runner]# kubectl create -f rbac-runner-config.yaml
serviceaccount/gitlab created
role.rbac.authorization.k8s.io/gitlab created
rolebinding.rbac.authorization.k8s.io/gitlab created
```

* 获取charts文件，并进行配置

```shell
[root@master gitlab-runner]# helm fetch gitlab/gitlab-runner						
[root@master gitlab-runner]# tar xf gitlab-runner-0.14.0.tgz
[root@master gitlab-runner]# vim gitlab-runner/values.yaml 
```

helm为我们提供了一个配置文件可以在安装 runner 的时候为其注册一个默认的 runner。我们可以去 [gitlab-runner 的项目源码](https://gitlab.com/gitlab-org/charts/gitlab-runner) 中获取到 `values.yaml` 这个配置文件。

#### 4.1.3 绑定docker.sock

由于在gitlab-runner中需要执行docker build 命令，需要docker服务端，我们可以来绑定K8s node节点的docker.sock来实现，编辑chart下面的template目录中的configmap.yaml，在大约42行修改

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200224231644.png)

```yaml
# add volume and bind docker.sock
cat >>/home/gitlab-runner/.gitlab-runner/config.toml <<EOF
  [[runners.kubernetes.volumes.pvc]]
    name = "{{.Values.maven.cache.pvcName}}"
    mount_path = "{{.Values.maven.cache.mountPath}}"
  [[runners.kubernetes.volumes.host_path]]
    name = "docker"
    mount_path = "/var/run/docker.sock"
EOF
```

#### 4.1.4 配置缓存

在我们的流程详解中能够看到生成的war/jar包制品需要存储在一个地方，这个地方就需要我们挂载一块外置的存储设备，在k8s中我们需要为其提供PVC，如果你有NFS存储或者分布式ceph制成的存储类都可以。

再此，我们机器中使用的为storageclass，演示利用存储类来声明PVC，在POD中进行挂载

* 创建pvc声明文件gitlab-runner-pvc.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-runner-pvc			# 创建名称为gitlab-runner-pvc的pvc
  namespace: gitlab-runners
spec:
  accessModes:
    - ReadWriteMany						# 访问模式
  storageClassName: rbd				# 存储类名称
  resources:
    requests:
      storage: 2Gi						# 存储大小
```

* 创建pvc

``` shell
[root@master gitlab-runner]# kubectl apply -f gitlab-runner-pvc.yaml                                         
persistentvolumeclaim/gitlab-runner-pvc created
[root@master gitlab-runner]# kubectl get pvc -n gitlab-runners       
NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
gitlab-runner-pvc   Bound    pvc-a620cd43-f396-4626-8599-c973c19cddd3   2Gi        RWO            rbd            4s
```

查看已经在gitlab-runners名称空间下已经创建好了PVC。

* 配置values

在上面我们配置绑定宿主机docker进程的时候也已经将pvc配置进去了，我们还需要在charts的values.yaml下面新添加配置 volume 的信息，并在 values.yaml 配置相应的变量

```
# 添加在values.yaml 的最后即可
maven:
  cache:
    pvcName: gitlab-runner-pvc
    mountPath: /home/cache/maven
```

#### 4.1.5 安装gitlab-runner

配置文件比较长，可以根据需要自己去配置，下面就贴下本文中需要配置的地方。

```shell
[root@master gitlab-runner]# egrep "^$|^#|^[[:space:]]+#" -v values.yaml 
imagePullPolicy: IfNotPresent
gitlabUrl: http://43.xx.xx.xx/												# gitlab url
runnerRegistrationToken: "nnxxxxxxxxxxxxxxxS"					# gitlab token
unregisterRunners: true
terminationGracePeriodSeconds: 3600
concurrent: 10
checkInterval: 30
rbac:
  create: true
  clusterWideAccess: false
  serviceAccountName: gitlab													# rbac的sa
metrics:
  enabled: true
runners:
  image: ubuntu:16.04																	# 使用的镜像
  tags: "gitlab-runner"
  privileged: true
  imagePullPolicy: "if-not-present"										# 如果执行具体job的runner不存在则拉取
  pollTimeout: 180
  outputLimit: 4096
  cache: {}
  builds: {}
  services: {}
  helpers: {}
  serviceAccountName: gitlab													# serviceaccount
  nodeSelector: {}

securityContext:
  fsGroup: 65533
  runAsUser: 100
resources: {}
affinity: {}
nodeSelector: {}
tolerations: []
hostAliases: []
podAnnotations: {}
podLabels: {}
maven:																								# 配置maven缓存信息
  cache:
    pvcName: gitlab-runner-pvc												# pvc信息
    mountPath: /home/cache/maven											# 挂载点
```

- 安装gitlab-runner

在配置完成values.yaml相关的字段后，我们利用helm来其 进行安装，后期如果我们有变动，直接修改value.yml ，然后进行更新即可`helm upgrade gitlab-rujner gitlab-runner/`

```yaml
[root@master gitlab-runner]# helm install --name gitlab-runner -f gitlab-runner/values.yaml --namespace gitlab-runners gitlab-runner/
NAME:   gitlab-runner
LAST DEPLOYED: Sun Feb 23 22:45:37 2020
NAMESPACE: gitlab-runners
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME                         DATA  AGE
gitlab-runner-gitlab-runner  5     1s

==> v1/Deployment
NAME                         READY  UP-TO-DATE  AVAILABLE  AGE
gitlab-runner-gitlab-runner  0/1    1           0          1s

==> v1/Pod(related)
NAME                                          READY  STATUS   RESTARTS  AGE
gitlab-runner-gitlab-runner-6745dd4cd6-r9z28  0/1    Pending  0         0s

==> v1/Secret
NAME                         TYPE    DATA  AGE
gitlab-runner-gitlab-runner  Opaque  2     1s


NOTES:

Your GitLab Runner should now be registered against the GitLab instance reachable at: "http://43.254.54.93:88/"
```

查看已经运行ok的相关pod资源，gitlab-runner的配置是通过configmap挂载进去，如果我们有新的配置可以修改，并删除之前的gitlab-runner的pod，其deployment控制器会进行创建Pod挂载新的配置文件。

- 查看pod运行状态

```shell
[root@master ~]# kubectl get po -n gitlab-runners 
NAME                                           READY   STATUS    RESTARTS   AGE
gitlab-runner-gitlab-runner-6745dd4cd6-r9z28   1/1     Running   0          21h
```

- 查看configmap及pod中挂载的配置

这个时候可以进入 pod 看一下 runner 的配置文件（`/home/gitlab-runner/.gitlab-runner/config.toml`）了。这个文件就是根据之前配置的 values.yaml 自动生成的。

```yaml
[root@master ~]# kubectl get cm -n gitlab-runners  gitlab-runner-gitlab-runner 
NAME                          DATA   AGE
gitlab-runner-gitlab-runner   5      21h
```

- 登录gitlab控制台查看目前gitlab-runner已经注册上了。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200225211019.png)

* 当pod运行起来后，查看完整配置

```yaml
[root@master gitlab-runner]# kubectl exec -it -n gitlab-runners gitlab-runner-gitlab-runner-6c7dfd859c-62dv5 -- cat /home/gitlab-runner/.gitlab-runner/config.toml
listen_address = "[::]:9252"
concurrent = 10
check_interval = 30
log_level = "info"

[session_server]
  session_timeout = 1800

[[runners]]
  name = "gitlab-runner-gitlab-runner-6c7dfd859c-62dv5"
  output_limit = 4096
  request_concurrency = 1
  url = "http://43.254.54.93:88/"									# 注册的url
  token = "eRskGn8Q-g4FZnZib63q"									# gitlab注册的token
  executor = "kubernetes"													# executor为kubernetes
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
  [runners.kubernetes]
    host = ""
    bearer_token_overwrite_allowed = false
    image = "ubuntu:16.04"												# 使用的镜像为ubuntu
    namespace = "gitlab-runners"									# 使用的namespace为gitlab-runners
    namespace_overwrite_allowed = ""
    privileged = true
    poll_timeout = 180
    service_account = "gitlab"										# serviceaccont 为gitlab
    service_account_overwrite_allowed = ""
    pod_annotations_overwrite_allowed = ""
    [runners.kubernetes.pod_security_context]
    [runners.kubernetes.volumes]
  [[runners.kubernetes.volumes.pvc]]
    name = "gitlab-runner-pvc"										# 挂载的pvc名称
    mount_path = "/home/cache/maven"							# 挂载在pod下的缓存目录
  [[runners.kubernetes.volumes.host_path]]
    name = "docker"
    mount_path = "/var/run/docker.sock"						# 绑定宿主机的docker.sock
```

至此我们已经将gitlab-runner在K8s集群中运行起来，接下来让我们来集成CI。

### 4.2 集成CI

#### 4.2.1 定义文件

由于使用的 executor 不同，所以. gitlab-ci.yml 和之前服务器也有些不同，不如image，默认如果每个stage中没有写指定某个image则使用该image

```yaml
image: docker:latest
variables:
  DOCKER_DRIVER: overlay2
  # k8s 挂载本地卷作为 maven 的缓存
  MAVEN_OPTS: "-Dmaven.repo.local=/home/cache/maven"

stages:
  - package							# 源码打包阶段
  - docker_build        # 镜像构建和打包推送阶段
  - deploy_k8s					# 应用部署阶段
  
before_script:
  - export APP_TAG="${CI_COMMIT_TAG:-${CI_COMMIT_SHA::8}}"				# 定义制作好的镜像tag

maven-package:
  #image: maven:3.5-jdk-8-alpine
  image: registry.cn-beijing.aliyuncs.com/codepipeline/public-blueocean-codepipeline-slave-java:0.1-63b99a20		# maven镜像进行java源码的build
  tags:
    - gitlab-runner				# 指定使用gitlab-runner来追寻
  stage: package
  script:
    - mvn clean package -Dmaven.test.skip=true
  artifacts:							# 将生成的war包上传到pvc挂载目录中
    paths:
      - target/*.war
docker-build:
  tags:
    - gitlab-runner
  stage: build
  script:
    - echo "Building Dockerfile-based application..."
    - docker build -t ${REGISTRY}/${QHUB_NAMESPACE}/${APP_NAME}:${APP_TAG} .					# 构建镜像
    - docker login --username=${DOCKER_USERNAME} ${REGISTRY} -p ${DOCKER_PASSWORD}		# 登录镜像仓
    - docker push ${REGISTRY}/${QHUB_NAMESPACE}/${APP_NAME}:${APP_TAG}								# push镜像
  only:
    - master
k8s-deploy:
  image: bitnami/kubectl:latest				# 使用kubectl镜像来进行最终的部署
  tags:
    - gitlab-runner
  stage: deploy
  script:
    - echo "deploy to k8s cluster..."
    - sed -i "s@$(grep -E "^[[:space:]]+image:" deployment.yaml | awk '{print $2}' |head -1)@${REGISTRY}/${QHUB_NAMESPACE}/${APP_NAME}:${APP_TAG}@g" 	deployment.yaml			# 替换deployment中的镜像
    - kubectl apply -f deployment.yaml						# 应用资源清单文件
  only:
    - master
```

可以看到CI文件中我们定义了三个stage：

1. 利用maven镜像进行打包，生成war包制品到缓存目录中。
2. 利用docker镜像对根据项目中的Dockerfile对缓存中的制品进行镜像构建，构建完成后登录镜像仓库进行镜像推送。
3. 利用kubectl镜像先对原有deployment文件进行镜像替换，之后进行将资源清单更新至K8s集群中。
4. 在为镜像打tag时，可以自行设置，也可以利用commit来进行与git版本对照。

#### 4.2.2 注意事项

- 由于在镜像构建的stage中，需要使用docker命令来进行相关操作，需要绑定本地 Docker 守护进程。
- maven 仓库的缓存，在又需要生产制品并保存的构建中，需要外部存储来存放制品。
- 在stage中引用的镜像最好能事先pull到各个node节点上，不然在第一次运行CI，如果网络有波动或带宽小，可能会因为镜像下载超时导致CI失败。

#### 4.2.3 配置变量

由于在CI文件中又一些敏感信息，例如镜像仓库的登录信息以及后期可以更改的镜像名称等，利用环境注入的方式，使得CI文件脱敏而且更具灵活性和适用性。

Project --> Settings --> CI/CD --> Variables， 添加GitLab Runner可用的环境变量，本示例添加变量如下：

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200225222659.png)

* APP_NAME：制作完成后镜像的名称

- DOCKER_USERNAME： 镜像仓库用户名
- DOCKER_PASSWORD 镜像仓库密码
- QHUB_NAMESPACE：镜像仓库的名称空间，在该项目中我们使用的腾讯的镜像仓库，当然可以自建harbor或使用其他公有云提供的镜像仓库或者dockerhub等。 
- REGISTRY：镜像仓库的地址

### 4.3 运行测试

在我们配置好了CI文件好，让我们来运行测试，提交代码，或者手动运行。

#### 4.3.1 运行pipeline

在项目CI/CD->Pipelines右上角有RunPipeline

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200225222632.png)

在图中我们可以看到在其中是可以并行运行多个pipeline，互不影响。

选择对应的分支或者传入变量手动运行pipeline

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200225222602.png)

#### 4.3.2 查看运行POD

登录K8s集群，查看此时运行的POD

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200225094722.png)

#### 4.3.3 查看Job

* 打包镜像

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200225222530.png)

* 镜像构建

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200225222504.png)

* 部署进K8s

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200225222420.png)

#### 4.3.4 查看构建情况

* 查看pipeline

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200225222353.png)

* 查看构建应用

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20200225222325.png)

至此我们就完成了示例Gitlab CI。

## 五 注意事项

* 在实战中的资源清单文件只写来deployment，读者可以根据自己的项目来，可以将service/configmap/hpa等资源清单文件也放在项目中托管，当然也可以制作成helm应用，每次发布利用helm来更新应用
* 在Gitlab CI中注意将敏感信息例如镜像仓库的登录方式等进行脱敏，将有变化的字段设置为变量，为此进行传入是的CI更扩展性及灵活性。
* 在此节中部分内容需要读者具备一定的K8s经验，例如其中的helm及相关的资源清单文件，如果前期了解后期业务微服务话后看本章节内容更为合适。
* 注意在写rbac与pvc的资源清单的名称空间为你想部署业务的名称空间，即例如你想把资源清单部署在common名称空间下，你就需要把pvc和rbac创建在common名称空间下，将gitlab-runner注册在common名称空间下。

## 六 应用场景

* 业务容器微服务化，或半微服务化
* 需要运维人员具有较强的K8s运维能力
* 追求极致持续集成持续交付体验gitops追崇者
* 有明确的流程规范，上线部署规范及明确可循的标准

## 七 反思

通过本次敏捷无敌之Gitlab CI实战，再次笔者感谢大家能够了解Gitlab的此特性，如果您的公司在使用Gitlab并且有使用其他的CI工具， 困于多套系统维护的复杂性，不妨尝试下Gitlab CI简单集成，Gitops的端到端运维工作，在云原生时代，将工具或中间件下沉到基础设施中，用户只需要专注于自身业务的开发,敏捷高效、文化中协作、快速试错、快速反馈、持续改进、不断迭代，以Git为来源打破研发与运维壁垒隔阂，实现产品更快、更频繁、更稳定的交付。








