## K8s可视化工具之KUR8

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220317124334.png)



## 一 KUR8简介

KUR8 是一个 Kubernetes 拓扑结构和 Prometheus 指标的可视化概览开源工具，只需要使用一个配置文件和 RBAC 授权的权限直接部署到你的 Kubernetes 集群中即可。KUR8 将在本地启动，让您一目了然地监控 Kubernetes 集群。



## 二 功能

### 2.1 可视化结构

**结构：**浏览 `Structure` 页面可以以轻松查看你的控制平面和工作节点及其所有 pod，单击组件可查看有关其元数据、状态和规范的更多详细信息，轻松查找有关从容器到入口的任何内容的镜像 ID 或 IP 地址的信息。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220317124620.png)

### 2.2 可视化集群状态

**指标：**使用我们精选的指标仪表板一目了然地了解集群的状态。



### 2.3 自定义指标

**自定义指标：**使用我们的自定义指标页面来使用 PROMQL 自动完成查询想要的任何指标

### 2.4 报警

**报警：**你的所有 Prometheus 报警都会显示在 `Alerts` 选项卡中，查明是否有任何警报正在触发以及它们属于哪些规则组。

## 三 部署

### 3.1 部署KUR8

K8s集群部署，由于应用需要读取K8s API，所以需要为期授予RBAC权限。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kur8-depl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kur8
  template:
    metadata:
      labels:
        app: kur8
    spec:
      containers:
        - name: kur8
          image: kur8/dashboard:latest
---
apiVersion: v1
kind: Service
metadata:
  name: kur8-srv
  labels:
    prometheus: cluster-monitoring
    k8s-app: kube-state-metrics
spec:
  selector:
    app: kur8
  type: ClusterIP
  ports:
    - name: kur8
      protocol: TCP
      port: 3000
      targetPort: 3000
      
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
 name: fabric8-rbac
subjects:
 - kind: ServiceAccount
   # Reference to upper's `metadata.name`
   name: default
   # Reference to upper's `metadata.namespace`
   namespace: default
roleRef:
 kind: ClusterRole
 name: cluster-admin
 apiGroup: rbac.authorization.k8s.io
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220317125856.png)

使用端口映射进行查看

```shell
kubectl port-forward deployment/kur8-depl 3068:3068
```

浏览器访问：[http://localhost:3068](http://localhost:3068/)

### 3.2 部署Prometheus

如果您没有安装 Prometheus 实例，请首先克隆存储库：

```
git clone https://github.com/oslabs-beta/KUR8
```

在 KUR8 目录中运行：

```
kubectl create -f infra/manifests/setup
```

设置完成后运行：

```
kubectl create -f infra/manifests/
```

如果您想将 Kur8 连接到 Prometheus，请通过以下方式打开端口：

```
kubectl --namespace monitoring port-forward svc/prometheus-k8s 9090
```

现在你就可以在 KUR8 中查看 Prometheus 选项卡，查看和创建您的自定义仪表板。

## 四 测试



![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220317142137.png)

## 参考链接

* https://github.com/oslabs-beta/KUR8#structure