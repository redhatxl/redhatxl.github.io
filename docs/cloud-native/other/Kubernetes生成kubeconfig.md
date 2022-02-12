# Kubernetes生成kubeconfig

# 一 背景

在公司中，为开发人员和团队创建隔离环境非常有用，尤其是对于培训。 如果您正在考虑 Kubernetes，您需要确保开发人员对它感到满意，并为他们提供一个安全的地方玩耍将有助于他们掌握该技术。

在微服务架构中尤其如此，您需要在隔离环境中测试您的应用程序，然后再将其发布给其他团队使用。 当工作负载太重而无法在单台笔记本电脑上运行时，它也很有用（例如：测试机器学习算法）。

在这篇文章中，我们将创建一个命名空间，然后使用 Kubernetes 的基于角色的访问控制 (RBAC) 系统创建一个只能访问该特定命名空间的服务帐户。 最后，我们将导出访问该命名空间所需的配置。

# 二 RBAC简介

RBAC里面的几种资源关系图

```shell
                  |--- Role --- RoleBinding                只在指定namespace中生效
ServiceAccount ---|
                  |--- ClusterRole --- ClusterRoleBinding  不受namespace限制，在整个K8s集群中生效
```

# 三 实战

## 3.1 创建ns

```shell
kubectl create namespace mynamespace

```

## 3.2 使用权限创建服务账户

```shell
cat > access.yaml<< EOF
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mynamespace-user
  namespace: mynamespace

---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: mynamespace-user-full-access
  namespace: mynamespace
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["batch"]
  resources:
  - jobs
  - cronjobs
  verbs: ["*"]

---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: mynamespace-user-view
  namespace: mynamespace
subjects:
- kind: ServiceAccount
  name: mynamespace-user
  namespace: mynamespace
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: mynamespace-user-full-access
EOF
```

如您所见，在 Role 定义中，我们添加了对该命名空间中所有内容的完全访问权限，包括诸如作业或 cronjob 之类的批处理类型。 因为它是一个角色，而不是一个 ClusterRole，所以它将被应用于单个命名空间：mynamespace。 有关 Kubernetes 中角色的更多详细信息，请查看官方文档。

```shell
kubectl create -f access.yaml

```

## 3.3 获取secret

创建sa用户后，系统会为sa创建对应的secret。

我们现在需要做的第一件事是获取服务帐户的秘密名称。运行以下命令并复制密钥的名称。

```shell
$ kubectl describe sa mynamespace-user -n mynamespace

Name:                mynamespace-user
Namespace:           mynamespace
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   mynamespace-user-token-tncrk
Tokens:              mynamespace-user-token-tncrk
Events:              <none>
$ skubectl get secret -n mynamespace
NAME                           TYPE                                  DATA   AGE
mynamespace-user-token-tncrk   kubernetes.io/service-account-token   3      16m
```

我们现在需要获取服务帐户的令牌和证书颁发机构。为此，我们将使用 kubectl 读取它们。现在，由于 Kubernetes 的秘密是 base64 编码的，我们还需要对它们进行解码

### 3.3.1 获取token

```shell
kubectl get secret mynamespace-user-token-xxxxx -n mynamespace -o "jsonpath={.data.token}" | base64 -D

kubectl get secret -n mynamespace mynamespace-user-token-tncrk -o "jsonpath={.data.token}" |base64 -D
eyJhbGciOiJSUzI1NiIsImtpZCI6Ik1fVDJTS1NhM0V1enlHTGFuN3BfNGZmOVM2bm9RTmdLZjlqWlpnbzA3ZEEifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJteW5hbWVzcGFjZSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJteW5hbWVzcGFjZS11c2VyLXRva2VuLXRuY3JrIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6Im15bmFtZXNwYWNlLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIzNTcxNDg4YS1mOTc5LTQ1YjMtOTE4ZS1jNjJkYmJhYzlmMjIiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6bXluYW1lc3BhY2U6bXluYW1lc3BhY2UtdXNlciJ9.oTghGOFPxyv0cJhQnrD7NdxPsil2JVZedJw5oIlHvlgY7B5ZMYbwhj9qd01GuZ5mjgiqKQJfndsf0fRziUR2TmgM4BQM-4MP8DJKG4eLW9zJx7pvrnFR-Ktf89AK-jHkmKg-yP7WS940NxeYctANh-sR4LJzJ-tRExNSOx54ZLW-dn4TuDo1pXj1DtOrHJsvhrP0CFaQWNTV1gDlucIKGo4dCU0LRiE1P1bgaHI4GBLTP2ez9VYtG24j9LLksvKWgWHu7zOKJlA2g1UDfgfrhu7dZltrhEbObLvu6hP57gSPSxH94ibSGAGhOWmAobqaxcKvGNqhbNO6KnmCjsFAqg%
```

### 3.3.2 获取ca

```shell
kubectl get secrets mynamespace-user-token-tncrk -n mynamespace  -o "jsonpath={.data['ca\.crt']}"
LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM1ekNDQWMrZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJeE1URXhOREF5TkRnME5sb1hEVE14TVRFeE1qQXlORGcwTmxvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBS2dPClpZdUdQWFVTc1ZWejBRSmRlTzNLK2JNQjl1TWNkc2xuTUlYdmI5Rmt3WjNCbjRNTHZaYUFrTC9RT2tNdkhEU1MKOTV4RElPTXRDZmJHWElKbEFJZ3YySUpTRUF6YmNNRE5hb2ZwZmpBVXVzUXd6TUhkdjVoRzRJbkg1UzRGdVFMaAp6Vm5jV1lFTDFORDFFZy9hWnMrTDFJemtGSHc1N1J3Q3hBY3dJcDY4azdLeFUyN24yOHYrVzVCY29HVWR0NGVoCkVaYVFOcGpNamRic3dHa1QwQVlLNFNWc1B2dDY5a2RsYlJld3gzYms5UEpYUFRqeWNkNmFMbUtDQk0yU1M0Q3EKbkJUM2NmS2l6ZGFza3VBTkFmWWQ4S0h4NE9rSXBMSHErM2JSeDltWDZYOUpXS1JIWVppM0VMZTVZa1NRcGQ3ZAo3ZXRKVmpsV0pFa3UwS3E5cW5VQ0F3RUFBYU5DTUVBd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZMK2Q0NDdvMmpPbXN4dmE5TG5wTHpWVjdoRlFNQTBHQ1NxR1NJYjMKRFFFQkN3VUFBNElCQVFCQlRnT2NPeVduWHJ5RTZ2YS9HNGlTU3c0MGZpNUZ2VnY0N0JMT0FBYUFjeURTNzFraQpnYmVERC9maUVSVkxWbi82Z1ZoektkaDRvMHZwaUxjNEZGejBhV1lhQlN6RVpnS1N5YzV5ejVxSndTNlJ4MjhSCmU5dEpsRUFWM3BYbXNTT3ppY2hRdVdiWkQ1NVFTT3ZXaEsvd3AveGxzR3ZEZSt4S0VMUmJTTGJmNzNCZzZvQ1gKTnZWWi9hSmovbk04WXhJOFZidzd1czZpK1FFMVRCVmpWZU1jSGlpTDByUlpnNkhEejY5THR4Qk1kRW9WQlJmbAp6TXdybE5laDJmbzFjTHpsRU1sVHIzUU93ZC9Rd0IyQ1MwcTZVOGRoZ1pVYkpTVmhWMHBCYVFDRmpLTS9jamFpCm96ajcvc2tUYjVhaDVqQVFqbFZBQi83cjlWVHk1U253eUg0QQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==%
```

## 3.4 创建kubeconfig

我们现在拥有所需的一切。剩下的唯一事情就是使用我们之前收集的数据创建 Kube 配置文件：

```shell
apiVersion: v1
kind: Config
preferences: {}

# Define the cluster
clusters:
- cluster:
    certificate-authority-data: PLACE CERTIFICATE HERE
    # You'll need the API endpoint of your Cluster here:
    server: https://YOUR_KUBERNETES_API_ENDPOINT
  name: my-cluster

# Define the user
users:
- name: mynamespace-user
  user:
    as-user-extra: {}
    client-key-data: PLACE CERTIFICATE HERE
    token: PLACE USER TOKEN HERE

# Define the context: linking a user to a cluster
contexts:
- context:
    cluster: my-cluster
    namespace: mynamespace
    user: mynamespace-user
  name: mynamespace

# Define current context
current-context: mynamespace
```

# 四 一键脚本

## 4.1 创建对指定namespace有所有权限的kube-config

```shell
#!/bin/bash
#
# This Script based on  https://jeremievallee.com/2018/05/28/kubernetes-rbac-namespace-user.html
# K8s'RBAC doc:         https://kubernetes.io/docs/reference/access-authn-authz/rbac
# Gitlab'CI/CD doc:     hhttps://docs.gitlab.com/ee/user/permissions.html#running-pipelines-on-protected-branches
#
# In honor of the remarkable Windson

BASEDIR="$(dirname "$0")"
folder="$BASEDIR/kube_config"

echo -e "All namespaces is here: \n$(kubectl get ns|awk 'NR!=1{print $1}')"
echo "endpoint server if local network you can use $(kubectl cluster-info |awk '/Kubernetes/{print $NF}')"

namespace=$1
endpoint=$(echo "$2" | sed -e 's,https\?://,,g')

if [[ -z "$endpoint" || -z "$namespace" ]]; then
    echo "Use "$(basename "$0")" NAMESPACE ENDPOINT";
    exit 1;
fi

if ! kubectl get ns|awk 'NR!=1{print $1}'|grep -w "$namespace";then kubectl create ns "$namespace";else echo "namespace: $namespace was exist." ;fi

echo "---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: $namespace-user
  namespace: $namespace
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: $namespace-user-full-access
  namespace: $namespace
rules:
- apiGroups: ['', 'extensions', 'apps', 'metrics.k8s.io']
  resources: ['*']
  verbs: ['*']
- apiGroups: ['batch']
  resources:
  - jobs
  - cronjobs
  verbs: ['*']
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: $namespace-user-view
  namespace: $namespace
subjects:
- kind: ServiceAccount
  name: $namespace-user
  namespace: $namespace
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: $namespace-user-full-access
---
# https://kubernetes.io/zh/docs/concepts/policy/resource-quotas/
apiVersion: v1
kind: ResourceQuota
metadata:
  name: $namespace-compute-resources
  namespace: $namespace
spec:
  hard:
    pods: "10"
    services: "10"
    persistentvolumeclaims: "5"
    requests.cpu: "1"
    requests.memory: 2Gi
    limits.cpu: "2"
    limits.memory: 4Gi" | kubectl apply -f -
kubectl -n $namespace describe quota $namespace-compute-resources
mkdir -p $folder
tokenName=$(kubectl get sa $namespace-user -n $namespace -o "jsonpath={.secrets[0].name}")
token=$(kubectl get secret $tokenName -n $namespace -o "jsonpath={.data.token}" | base64 --decode)
certificate=$(kubectl get secret $tokenName -n $namespace -o "jsonpath={.data['ca\.crt']}")

echo "apiVersion: v1
kind: Config
preferences: {}
clusters:
- cluster:
    certificate-authority-data: $certificate
    server: https://$endpoint
  name: $namespace-cluster
users:
- name: $namespace-user
  user:
    as-user-extra: {}
    client-key-data: $certificate
    token: $token
contexts:
- context:
    cluster: $namespace-cluster
    namespace: $namespace
    user: $namespace-user
  name: $namespace
current-context: $namespace" > $folder/$namespace.kube.conf
```

## 4.2 创建对指定namespace有所有权限的kube-config（在已有的namespace中创建）

```shell
#!/bin/bash


BASEDIR="$(dirname "$0")"
folder="$BASEDIR/kube_config"

echo -e "All namespaces is here: \n$(kubectl get ns|awk 'NR!=1{print $1}')"
echo "endpoint server if local network you can use $(kubectl cluster-info |awk '/Kubernetes/{print $NF}')"

namespace=$1
endpoint=$(echo "$2" | sed -e 's,https\?://,,g')

if [[ -z "$endpoint" || -z "$namespace" ]]; then
    echo "Use "$(basename "$0")" NAMESPACE ENDPOINT";
    exit 1;
fi


echo "---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: $namespace-user
  namespace: $namespace
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: $namespace-user-full-access
  namespace: $namespace
rules:
- apiGroups: ['', 'extensions', 'apps', 'metrics.k8s.io']
  resources: ['*']
  verbs: ['*']
- apiGroups: ['batch']
  resources:
  - jobs
  - cronjobs
  verbs: ['*']
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: $namespace-user-view
  namespace: $namespace
subjects:
- kind: ServiceAccount
  name: $namespace-user
  namespace: $namespace
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: $namespace-user-full-access" | kubectl apply -f -

mkdir -p $folder
tokenName=$(kubectl get sa $namespace-user -n $namespace -o "jsonpath={.secrets[0].name}")
token=$(kubectl get secret $tokenName -n $namespace -o "jsonpath={.data.token}" | base64 --decode)
certificate=$(kubectl get secret $tokenName -n $namespace -o "jsonpath={.data['ca\.crt']}")

echo "apiVersion: v1
kind: Config
preferences: {}
clusters:
- cluster:
    certificate-authority-data: $certificate
    server: https://$endpoint
  name: $namespace-cluster
users:
- name: $namespace-user
  user:
    as-user-extra: {}
    client-key-data: $certificate
    token: $token
contexts:
- context:
    cluster: $namespace-cluster
    namespace: $namespace
    user: $namespace-user
  name: $namespace
current-context: $namespace" > $folder/$namespace.kube.conf
```

# 参考链接

* https://www.toutiao.com/i6942467217019666952?wid=1639197385733
* https://jeremievallee.com/2018/05/28/kubernetes-rbac-namespace-user.html