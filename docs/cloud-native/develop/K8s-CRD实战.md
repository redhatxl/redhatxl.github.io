# K8s-CRD 实战

# 一 概述

## 1.1 CRD简介

Custom resources：是对K8S API的扩展，代表了一个特定的kubetnetes的定制化安装。在一个运行中的集群中，自定义资源可以动态注册到集群中。注册完毕以后，用户可以通过kubelet创建和访问这个自定义的对象，类似于操作pod一样。

Custom controllers：Custom resources可以让用户简单的存储和获取结构化数据。只有结合控制器才能变成一个真正的declarative API（被声明过的API）。控制器可以把资源更新成用户想要的状态，并且通过一系列操作维护和变更状态。定制化控制器是用户可以在运行中的集群内部署和更新的一个控制器，它独立于集群本身的生命周期。
定制化控制器可以和任何一种资源一起工作，当和定制化资源结合使用时尤其有效。

Operator模式 是一个customer controllers和Custom resources结合的例子。它可以允许开发者将特殊应用编码至kubernetes的扩展API内。



## 1.2 使用场景

其实crd在很多k8s周边开源项目中有使用，比如ingress-controller 、istio 、hpa和众多的operator。

如何添加一个Custom resources到我的kubernetes集群呢？ kubernetes提供了两种方式将Custom resources添加到集群。

Custom Resource Definitions (CRDs)：更易用、不需要编码。但是缺乏灵活性。
API Aggregation：需要编码，允许通过聚合层的方式提供更加定制化的实现。
本文重点讲解Custom Resource Definitions (CRD）的使用方法。

## 1.3 通俗讲解

* 在k8s中，所有的东西都叫做资源（Resource）， 也就是yaml中Kind字段
* 除过K8s集群中常见的deployment这些资源外，kube允许用户自定义资源（Custom Resource），也就CR
* CRD 其实并不是自定义资源，而是我们自定义资源的定义（来描述我们定义的资源是什么样子）
* 对于一个CRD来讲，其本质就是一个 Open Api 的 schema，无论是k8s自由资源，还是用户自定义资源，都需要yaml来描述，但是如果保障yaml的规范性和合法性，就是schema来做，就是想集群注册一种新资源，并告知 ApiServer，这种资源怎么怎么被合法的定义。

# 二 架构

## 2.1 架构图



## 2.2 架构详解



当您创建一个新的 CustomResourceDefinition (CRD)时，Kubernetes API Server 将为您指定的每个版本创建一个新的 RESTful 资源路径。CRD 可以是命名空间的，也可以是集群范围的，这在 CRD 的范围字段中指定了。与现有的内置对象一样，删除名称空间将删除该名称空间中的所有自定义对象。Customresourcedefinition 本身是非命名空间的，对所有命名空间都可用。

# 三 实战

## 3.1 环境



## 3.2 自定义资源

### 3.2.1 crontabl_crd.yaml

* 内容

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # 称必须与下面的spec字段匹配，格式为: <plural>.<group>
  name: crontabs.crd.test.com
spec:
  # 用于REST API的组名称: /apis/<group>/<version>
  group: crd.test.com
  versions:
  - name: v1
    # 每个版本都可以通过服务标志启用/禁用。
    served: true
    # 必须将一个且只有一个版本标记为存储版本。
    storage: true
  scope: Namespaced  # 指定crd资源作用范围在命名空间或集群
  names:
    # URL中使用的复数名称: /apis/<group>/<version>/<plural>
    plural: crontabs
    # 在CLI(shell界面输入的参数)上用作别名并用于显示的单数名称
    singular: crontab
    kind: CronTab
    # 短名称允许短字符串匹配CLI上的资源，意识就是能通过kubectl 在查看资源的时候使用该资源的简名称来获取。
    shortNames:
    - ct
```

* 部署crd

```shell
[root@master concrd]# kubectl apply -f crontab_crd.yaml 
customresourcedefinition.apiextensions.k8s.io/crontabs.crd.test.com created
[root@master concrd]# kubectl get crd |grep crontab
crontabs.crd.test.com                             2021-05-19T06:42:02Z
```

然后在以下地方创建一个新的名称空间 RESTful API 端点:

```
/apis/stable.example.com/v1/namespaces/*/crontabs/...
```

### 3.2.2 test_crontab.yaml

* 内容

```yaml
apiVersion: crd.test.com/v1
kind: CronTab
metadata:
  name: my-test-crontab
spec:
  cronSpec: "* * * * */10"
  image: my-test-image
  replicas: 2
```

* 部署

```shell
[root@master concrd]# kubectl apply -f test_crontabl.yaml 
crontab.crd.test.com/my-test-crontab created
[root@master concrd]# kubectl get ct
NAME              AGE
my-test-crontab   6s
[root@master concrd]# kubectl  get ct my-test-crontab -oyaml
apiVersion: crd.test.com/v1
kind: CronTab
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"crd.test.com/v1","kind":"CronTab","metadata":{"annotations":{},"name":"my-test-crontab","namespace":"default"},"spec":{"cronSpec":"* * * * */10","image":"my-test-image","replicas":2}}
  creationTimestamp: "2021-05-19T06:44:26Z"
  generation: 1
  name: my-test-crontab
  namespace: default
  resourceVersion: "67685692"
  selfLink: /apis/crd.test.com/v1/namespaces/default/crontabs/my-test-crontab
  uid: fc78e12d-6b0f-4dc5-b6a7-75459d333eef
spec:
  cronSpec: '* * * * */10'
  image: my-test-image
  replicas: 2
```

## 3.3 validations

validation这个验证是为了在创建好自定义资源后，通过该资源创建对象的时候，对象的字段中存在无效值，则创建该对象的请求将被拒绝，否则会被创建。我们可以在crd文件中添加“validation:”字段来添加相应的验证机制。

### 3.3.1 crontab_validations_crd.yaml

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # 称必须与下面的spec字段匹配，格式为: <plural>.<group>
  name: crontabs.crd.test.com
spec:
  # 用于REST API的组名称: /apis/<group>/<version>
  group: crd.test.com
  versions:
  - name: v1
    # 每个版本都可以通过服务标志启用/禁用。
    served: true
    # 必须将一个且只有一个版本标记为存储版本。
    storage: true
  scope: Namespaced # 指定crd资源作用范围在命名空间或集群
  names:
    # URL中使用的复数名称: /apis/<group>/<version>/<plural>
    plural: crontabs
    # 在CLI(shell界面输入的参数)上用作别名并用于显示的单数名称
    singular: crontab
    kind: CronTab
    # 短名称允许短字符串匹配CLI上的资源，意识就是能通过kubectl 在查看资源的时候使用该资源的简名称来获取。
    shortNames:
    - ct
  validation:
    openAPIV3Schema:
      properties:
        spec:
          properties:
            cronSpec: #必须是字符串、符合正则规则
              type: string
              pattern: '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
            replicas: #设置副本数的限制
              type: integer
              minimum: 1
              maximum: 10

```

### 3.3.2 test_validations_crontab.yaml

```yaml
apiVersion: crd.test.com/v1
kind: CronTab
metadata:
  name: my-test-crontab
spec:
  cronSpec: "* * * * */10"
  image: my-test-image
  replicas: 21
```

* 测试不符合规范的replicase

```shell
[root@master concrd]# kubectl apply -f test_validation_crontab.yaml 
The CronTab "my-test-crontab" is invalid: spec.replicas: Invalid value: 10: spec.replicas in body should be less than or equal to 10
```

## 3.4 additionalPrinterColumns

从Kubernetes 1.11开始，kubectl使用服务器端打印。服务器决定由kubectl get命令显示哪些列即在我们获取一个内置资源的时候会显示出一些列表信息(比如：kubectl get nodes)。这里我们可以使用CustomResourceDefinition自定义这些列，当我们在查看自定义资源信息的时候显示出我们需要的列表信息。通过在crd文件中添加“additionalPrinterColumns:”字段，在该字段下声明需要打印列的的信息

### 3.4.1 crontab_crd_additionalPrinterColumns.yaml

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # 称必须与下面的spec字段匹配，格式为: <plural>.<group>
  name: crontabs.crd.test.com
spec:
  # 用于REST API的组名称: /apis/<group>/<version>
  group: crd.test.com
  versions:
  - name: v1
    # 每个版本都可以通过服务标志启用/禁用。
    served: true
    # 必须将一个且只有一个版本标记为存储版本。
    storage: true
  scope: Namespaced # 指定crd资源作用范围在命名空间或集群
  names:
    # URL中使用的复数名称: /apis/<group>/<version>/<plural>
    plural: crontabs
    # 在CLI(shell界面输入的参数)上用作别名并用于显示的单数名称
    singular: crontab
    kind: CronTab
    # 短名称允许短字符串匹配CLI上的资源，意识就是能通过kubectl 在查看资源的时候使用该资源的简名称来获取。
    shortNames:
    - ct
  validation:
    openAPIV3Schema:
      properties:
        spec:
          properties:
            cronSpec: #必须是字符串、符合正则规则
              type: string
              pattern: '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
            replicas: #设置副本数的限制
              type: integer
              minimum: 1
              maximum: 10
  additionalPrinterColumns:
  - name: Replicas
    type: integer
    JSONPath: .spec.replicas
  - name: Age
    type: date
    JSONPath: .metadata.creationTimestamp

```

* 部署

```shell
[root@master concrd]# kubectl apply -f crontab_crd_additionalPrinterColumns.yaml 
customresourcedefinition.apiextensions.k8s.io/crontabs.crd.test.com created
[root@master concrd]# kubectl get crd |grep crontab
crontabs.crd.test.com                             2021-05-19T06:42:02Z
```

### 3.4.2 test_additionalPrinterColumns_crontab.yml

* 内容

```yaml
apiVersion: crd.test.com/v1
kind: CronTab
metadata:
  name: my-test-crontab
spec:
  cronSpec: "* * * * */10"
  image: my-test-image
  replicas: 2


```

* 查看

```shell
[root@master concrd]# kubectl  get ct
NAME              REPLICAS   AGE
my-test-crontab   2          178m
```



## 3.5 subresources

一般我们要是没有在自定义资源当中配置关于资源对象的伸缩和状态信息的一些相关配置的话，那么在当我们通过该自定义资源创建对象后，又想通过“kubectl scale”来弹性的扩展该对象的容器的时候就会无能为力。而CRD可以允许我们添加该方面的相关配置声明，从而达到我们可以对自定义资源对象的伸缩处理。添加“ subresources:”字段来声明状态和伸缩信息。

### 3.5.1 crontab_crd_subresources.yml

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # 称必须与下面的spec字段匹配，格式为: <plural>.<group>
  name: crontabs.crd.test.com
spec:
  # 用于REST API的组名称: /apis/<group>/<version>
  group: crd.test.com
  versions:
  - name: v1
    # 每个版本都可以通过服务标志启用/禁用。
    served: true
    # 必须将一个且只有一个版本标记为存储版本。
    storage: true
  scope: Namespaced # 指定crd资源作用范围在命名空间或集群
  names:
    # URL中使用的复数名称: /apis/<group>/<version>/<plural>
    plural: crontabs
    # 在CLI(shell界面输入的参数)上用作别名并用于显示的单数名称
    singular: crontab
    kind: CronTab
    # 短名称允许短字符串匹配CLI上的资源，意识就是能通过kubectl 在查看资源的时候使用该资源的简名称来获取。
    shortNames:
    - ct
  validation:
    openAPIV3Schema:
      properties:
        spec:
          properties:
            cronSpec: #必须是字符串、符合正则规则
              type: string
              pattern: '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
            replicas: #设置副本数的限制
              type: integer
              minimum: 1
              maximum: 10
  additionalPrinterColumns:
  - name: Replicas
    type: integer
    JSONPath: .spec.replicas
  - name: Age
    type: date
    JSONPath: .metadata.creationTimestamp
  subresources:
    scale:
      specReplicasPath: .spec.replicas
      statusReplicasPath: .status.replicas

```





### 3.5.2 测试

```shell
[root@master concrd]# kubectl  apply -f crontab_crd_subresources.yml 
customresourcedefinition.apiextensions.k8s.io/crontabs.crd.test.com configured
[root@master concrd]# kubectl  get ct
NAME              REPLICAS   AGE
my-test-crontab   2          3h1m
[root@master concrd]# kubectl scale --replicas=5 crontabs/my-test-crontab
crontab.crd.test.com/my-test-crontab scaled
[root@master concrd]# kubectl  get ct
NAME              REPLICAS   AGE
my-test-crontab   5          3h1m
```

##  清理

* 删除自定义资源

```shell
kubectl delete  ct my-test-crontab
```

* 清理crd

```shell
kubectl delete crd crontabs.crd.test.com
```



# 四 注意事项







# 五 总结

功能上：CRD 使得 Kubernetes 已有的资源和能力变成了乐高积木，我们很轻松就可以利用这些积木拓展 Kubernetes 原生不具备的能力。

产品上：基于 Kubernetes 做的产品无法避免的需要让我们将产品术语向 Kube 术语靠拢，比如一个服务就是一个 Deployment，一个实例就是一个 Pod 之类。但是 CRD 允许我们自己基于产品创建概念（或者说资源），让 Kube 已有的资源为我们的概念服务，这可以使产品更专注与解决的场景，而不是如何思考如何将场景应用到 Kubernetes。





# 参考链接

* https://blog.csdn.net/weixin_41806245/article/details/94451734
* https://juejin.cn/post/6844903965482713096
* https://juejin.cn/post/6844904199029784589
* https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/
* https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#validation