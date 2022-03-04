# Kubernetes安全扫描之kubescape

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220301221857.png)

## 一 背景

Kubescape 是第一个用于测试 Kubernetes 是否按照 NSA 和 CISA 的 Kubernetes 强化指南中定义的安全部署的工具 使用 Kubescape 测试集群或扫描单个 YAML 文件并将其集成到您的流程中。

## 二 特性

* 功能：提供多云 K8s 集群检测，包括风险分析、安全性兼容、 RBAC 可视化工具和图像漏洞扫描。

* 集群扫描：Kubescape 扫描 K8s 集群、 YAML 文件和 HELM 图表，根据多个框架(如 NSA-CISA、 MITRE att & ck)、软件漏洞和在 CI/CD 管道早期阶段的 RBAC (基于角色的访问控制)违规检测错误配置，即时计算风险评分并显示随时间推移的风险趋势。

* 工具集成：与其他 DevOps 工具集成，包括 Jenkins、 CircleCI、 Github 工作流、 Prometheus 和 Slack，并支持多云 k8部署，如 EKS、 GKE 和 AKS。

由于其简单易用的 CLI 界面、灵活的输出格式和自动扫描能力，它成为开发人员中增长最快的 Kubernetes 工具之一，节省了用户和管理员宝贵的时间、精力和资源

## 三 安装

### 3.1 Mac 安装

```shell
  brew tap armosec/kubescape
  brew install kubescape
```

### 3.2 Linux安装

```shell
curl -s https://raw.githubusercontent.com/armosec/kubescape/master/install.sh | /bin/bash
```

## 四 集群检测

### 4.1 执行扫描

扫描可分为本地集群扫描和提交结果到saas平台。

```shell
chmod +x /root/.kubescape/kubescape 
/root/.kubescape/kubescape scan framework nsa --exclude-namespaces kube-system,kube-public
```

### 4.2 输出结果

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210918094520.png)

## 五 SaaS平台

saas平台地址：https://portal.armo.cloud/，主要有三大功能，

* 集群检测
* 镜像扫描
* RBAC可视化

### 5.1 集群检测

与单独在集群检测输出不同，saas平台会将集群检测及上传到平台进行可视化展示

```shell
curl -s https://raw.githubusercontent.com/armosec/kubescape/master/install.sh | /bin/bash
```

* 提交检测结果到平台

```shell
export ID=6xxxxxxxxxxx
kubescape scan --submit --account=${ID}
```

* 查看结果

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220302095013.png)

同时可根据提示跳转到详细安全信息内容

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220302095047.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220302102952.png)

### 5.2 镜像扫描

```shell
helm repo add armo https://armosec.github.io/armo-helm/

helm upgrade --install armo  armo/armo-cluster-components -n armo-system --create-namespace --set accountGuid=${ID} --set clusterName=`kubectl config current-context`
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220302100140.png)

* 查看结果

可以看到集群整个镜像扫描的信息。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220302102430.png)

同时针对单个镜像也可以看到详细扫描信息。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220302102538.png)

可以针对单个镜像可以看是否有FIX AVAILABLE，以及FIX IN VERSION

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220302103622.png)



### 5.3 RBAC可视化

目前RBAC功能还处于beat阶段。

## 六 其他

kubescape可以很方便的在Node节点对集群进行包风险分析、安全性兼容、 RBAC 可视化工具和图像漏洞扫描，同时可以利用SaaS非常方便的进行可视化，同时可以非常方便的与其他系统进行集成。

## 参考链接

* https://github.com/armosec/kubescape
* https://portal.armo.cloud/