# Kubernetes AdmissionWebhook

# 前言

Kubernetes 对 API 访问提供了三种安全访问控制措施：认证、授权和 Admission Control。认证解决用户是谁的问题，授权解决用户能做什么的问题，Admission Control 则是资源管理方面的作用。通过合理的权限管理，能够保证系统的安全可靠。

本文主要讲讲Admission中ValidatingAdmissionWebhook和MutatingAdmissionWebhook。

# 一 AdmissionWebhook简介

我们知道k8s在各个方面都具备可扩展性，比如通过cni实现多种网络模型，通过csi实现多种存储引擎，通过cri实现多种容器运行时等等。而AdmissionWebhook就是另外一种可扩展的手段。 除了已编译的Admission插件外，可以开发自己的Admission插件作为扩展，并在运行时配置为webhook。

Admission webhooks是HTTP回调，它接收Admission请求并对它们做一些事情。可以定义两种类型的Admission webhook，ValidatingAdmissionWebhook和MutatingAdmissionWebhook。

如果启用了MutatingAdmission，当开始创建一种k8s资源对象的时候，创建请求会发到你所编写的controller中，然后我们就可以做一系列的操作。比如我们的场景中，我们会统一做一些功能性增强，当业务开发创建了新的deployment，我们会执行一些注入的操作，比如敏感信息aksk，或是一些优化的init脚本。

而与此类似，只不过ValidatingAdmissionWebhook 是按照你自定义的逻辑是否允许资源的创建。比如，我们在实际生产k8s集群中，处于稳定性考虑，我们要求创建的deployment 必须设置request和limit。

## 1.1 **什么是AdmissionWebhook**



什么是AdmissionWebhook，就要先了解K8S中的admission controller, 按照官方的解释是： admission controller是拦截(经过身份验证)API Server请求的网关，并且可以修改请求对象或拒绝请求。 

简而言之，它可以认为是拦截器，类似web框架中的middleware。

K8S默认提供很多内置的admission controller，通过kube-apiserver启动命令参数可以 查看到支持的admission controller plugin有哪些。

```javascript
[root@node220]# kube-apiserver --help |grep enable-admission-plugins

# 支持的plugin有如下
AlwaysAdmit, AlwaysDeny, AlwaysPullImages, 
DefaultStorageClass, DefaultTolerationSeconds, DenyEscalatingExec, 
DenyExecOnPrivileged, EventRateLimit, ExtendedResourceToleration, 
ImagePolicyWebhook, Initializers, LimitPodHardAntiAffinityTopology, 
LimitRanger, MutatingAdmissionWebhook, NamespaceAutoProvision, 
NamespaceExists, NamespaceLifecycle, NodeRestriction, 
OwnerReferencesPermissionEnforcement, PersistentVolumeClaimResize, 
PersistentVolumeLabel, PodNodeSelector, PodPreset, PodSecurityPolicy, 
PodTolerationRestriction, Priority, ResourceQuota, SecurityContextDeny, 
ServiceAccount, StorageObjectInUseProtection, ValidatingAdmissionWebhook. 
```

这里enable的admission-plugins如下

```javascript
--enable-admission-plugins=PersistentVolumeClaimResize,NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction,ValidatingAdmissionWebhook,MutatingAdmissionWebhook 
```

这里不对每个plugin详细说明，网上都可以搜到相关资料。 总体来说，admission-plugins分为三大类：

1.修改类型(mutating)

2.验证类型(validating)

3.既是修改又是验证类型(mutating&validating)

这些admission plugin构成一个顺序链，先后顺序决定谁先调用，但不会影响使用。

这里关心的plugin有**两个**：

- MutatingAdmissionWebhook: 做修改操作的AdmissionWebhook 
- ValidatingAdmissionWebhook: 做验证操作的AdmissionWebhook

引用kubernetes官方博客的一张图来说明MutatingAdmissionWebhook和ValidatingAdmissionWebhook所处的位置:

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210922135242.png)

解释下这个过程：

1.api请求到达K8S API Server

 2.请求要先经过认证

- kubectl调用需要kubeconfig
- 直接调用K8S api需要证书+bearToken
- client-go调用也需要kubeconfig

3.执行一连串的admission controller，包括MutatingAdmissionWebhook和ValidatingAdmissionWebhook, 先串行执行MutatingAdmission的Webhook list 4.对请求对象的schema进行校验 5.并行执行ValidatingAdmission的Webhook list 6.最后写入etcd

## 1.2 **Initializers vs AdmissionWebhook区别**

二者都能实现动态可扩展载入admission controller， Initializers是串行执行，在高并发场景容易导致对象停留在uninitialized状态，影响继续调度。 Alpha Initializers特性在k8s 1.14版本被移除了，详情见https://github.com/kubernetes/apimachinery/issues/60。 相比Initializers，官方更推荐AdmissionWebhook；MutatingAdmissionWebhook是串行执行，ValidatingAdmissionWebhook是并行执行，性能更好。

# 二 **AdmissionWebhook应用场景**

Istio相信大家都有听过，Istio就是采用AdmissionWebhook实现sidecar容器自动注入。我目前用到的应用场景有两个，当然肯定还有其它应用场景。

- 自动打标签 比如启动一个应用，应用包括deployment、service、ingress； 怎么快速过滤出哪些资源属于应用? 在K8S中，pod、service、ingress 都是独立的资源，通过给这些资源打上label，是最快速的方式。
- 自动注入sidecar容器 应用启动后，应用的监控、日志如何处理？借助sidecar容器注入到其pod中

收集应用日志的sidecar容器可以像下图所示，应用监控的sidecar容器类似

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210922135351.png)











# 实战开发admissionwebhook

### 前提条件

- k8s版本需至少v1.9
- 确保启用了 MutatingAdmissionWebhook and ValidatingAdmissionWebhook admission controllers
- 确定 启用了[http://admissionregistration.k8s.io/v1beta1](https://link.zhihu.com/?target=http%3A//admissionregistration.k8s.io/v1beta1)

### 写一个 admission webhook server

官方提供了有个[demo](https://link.zhihu.com/?target=https%3A//github.com/kubernetes/kubernetes/blob/v1.13.0/test/images/webhook/main.go)。大家可以详细研究，核心思想就是：
webhook处理apiservers发送的AdmissionReview请求，并将其决定作为AdmissionReview对象发送回去。

```go
package main

import (
    "encoding/json"
    "flag"
    "fmt"
    "io/ioutil"
    "net/http"

    "k8s.io/api/admission/v1beta1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/klog"
    // TODO: try this library to see if it generates correct json patch
    // https://github.com/mattbaird/jsonpatch
)

// toAdmissionResponse is a helper function to create an AdmissionResponse
// with an embedded error
func toAdmissionResponse(err error) *v1beta1.AdmissionResponse {
    return &v1beta1.AdmissionResponse{
        Result: &metav1.Status{
            Message: err.Error(),
        },
    }
}

// admitFunc is the type we use for all of our validators and mutators
type admitFunc func(v1beta1.AdmissionReview) *v1beta1.AdmissionResponse

// serve handles the http portion of a request prior to handing to an admit
// function
func serve(w http.ResponseWriter, r *http.Request, admit admitFunc) {
    var body []byte
    if r.Body != nil {
        if data, err := ioutil.ReadAll(r.Body); err == nil {
            body = data
        }
    }

    // verify the content type is accurate
    contentType := r.Header.Get("Content-Type")
    if contentType != "application/json" {
        klog.Errorf("contentType=%s, expect application/json", contentType)
        return
    }

    klog.V(2).Info(fmt.Sprintf("handling request: %s", body))

    // The AdmissionReview that was sent to the webhook
    requestedAdmissionReview := v1beta1.AdmissionReview{}

    // The AdmissionReview that will be returned
    responseAdmissionReview := v1beta1.AdmissionReview{}

    deserializer := codecs.UniversalDeserializer()
    if _, _, err := deserializer.Decode(body, nil, &requestedAdmissionReview); err != nil {
        klog.Error(err)
        responseAdmissionReview.Response = toAdmissionResponse(err)
    } else {
        // pass to admitFunc
        responseAdmissionReview.Response = admit(requestedAdmissionReview)
    }

    // Return the same UID
    responseAdmissionReview.Response.UID = requestedAdmissionReview.Request.UID

    klog.V(2).Info(fmt.Sprintf("sending response: %v", responseAdmissionReview.Response))

    respBytes, err := json.Marshal(responseAdmissionReview)
    if err != nil {
        klog.Error(err)
    }
    if _, err := w.Write(respBytes); err != nil {
        klog.Error(err)
    }
}

func serveAlwaysDeny(w http.ResponseWriter, r *http.Request) {
    serve(w, r, alwaysDeny)
}

func serveAddLabel(w http.ResponseWriter, r *http.Request) {
    serve(w, r, addLabel)
}

func servePods(w http.ResponseWriter, r *http.Request) {
    serve(w, r, admitPods)
}

func serveAttachingPods(w http.ResponseWriter, r *http.Request) {
    serve(w, r, denySpecificAttachment)
}

func serveMutatePods(w http.ResponseWriter, r *http.Request) {
    serve(w, r, mutatePods)
}

func serveConfigmaps(w http.ResponseWriter, r *http.Request) {
    serve(w, r, admitConfigMaps)
}

func serveMutateConfigmaps(w http.ResponseWriter, r *http.Request) {
    serve(w, r, mutateConfigmaps)
}

func serveCustomResource(w http.ResponseWriter, r *http.Request) {
    serve(w, r, admitCustomResource)
}

func serveMutateCustomResource(w http.ResponseWriter, r *http.Request) {
    serve(w, r, mutateCustomResource)
}

func serveCRD(w http.ResponseWriter, r *http.Request) {
    serve(w, r, admitCRD)
}

func main() {
    var config Config
    config.addFlags()
    flag.Parse()

    http.HandleFunc("/always-deny", serveAlwaysDeny)
    http.HandleFunc("/add-label", serveAddLabel)
    http.HandleFunc("/pods", servePods)
    http.HandleFunc("/pods/attach", serveAttachingPods)
    http.HandleFunc("/mutating-pods", serveMutatePods)
    http.HandleFunc("/configmaps", serveConfigmaps)
    http.HandleFunc("/mutating-configmaps", serveMutateConfigmaps)
    http.HandleFunc("/custom-resource", serveCustomResource)
    http.HandleFunc("/mutating-custom-resource", serveMutateCustomResource)
    http.HandleFunc("/crd", serveCRD)
    server := &http.Server{
        Addr:      ":443",
        TLSConfig: configTLS(config),
    }
    server.ListenAndServeTLS("", "")
}
```

### 动态配置admission webhooks

您可以通过[ValidatingWebhookConfiguration](https://link.zhihu.com/?target=https%3A//kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/%23validatingwebhookconfiguration-v1beta1-admissionregistration-k8s-io)或[MutatingWebhookConfiguration](https://link.zhihu.com/?target=https%3A//kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/%23mutatingwebhookconfiguration-v1beta1-admissionregistration-k8s-io)动态配置哪些资源受入口webhooks的限制。

具体示例如下：

```yaml
apiVersion: admissionregistration.k8s.io/v1beta1
kind: ValidatingWebhookConfiguration
metadata:
  name: <name of this configuration object>
webhooks:
- name: <webhook name, e.g., pod-policy.example.io>
  rules:
  - apiGroups:
    - ""
    apiVersions:
    - v1
    operations:
    - CREATE
    resources:
    - pods
    scope: "Namespaced"
  clientConfig:
    service:
      namespace: <namespace of the front-end service>
      name: <name of the front-end service>
    caBundle: <pem encoded ca cert that signs the server cert used by the webhook>
  admissionReviewVersions:
  - v1beta1
  timeoutSeconds: 1
```



# 总结

最后我们来总结下 webhook Admission 的优势：

- webhook 可动态扩展 Admission 能力，满足自定义客户的需求
- 不需要重启 API Server，可通过创建 webhook configuration 热加载 webhook admission

# 参考链接

* https://banzaicloud.com/blog/k8s-admission-webhooks/
* https://cloud.tencent.com/developer/article/1445760
* https://github.com/kubernetes/kubernetes/blob/v1.13.0/test/images/webhook/main.go
* https://zhuanlan.zhihu.com/p/136173524