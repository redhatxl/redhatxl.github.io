# ksdevelop

## 一 拉取代码

```shell
cd /Users/xuel/workspace/goworkspace/src/github.com/redhatxl
cd kubesphere
git clone https://github.com/redhatxl/kubesphere.git 
git remote add upstream https://github.com/kubesphere/kubesphere.git

git remote set-url --push upstream no_push
git remote -v       

origin  https://github.com/redhatxl/kubesphere.git (fetch)
origin  https://github.com/redhatxl/kubesphere.git (push)
upstream        https://github.com/kubesphere/kubesphere.git (fetch)
upstream        no_push (push)

```

## 二 数据结构

```go
.
├── CONTRIBUTING.md
├── LICENSE
├── Makefile
├── OWNERS
├── README.md
├── README_zh.md
├── api						// api 文档
│   ├── api-rules
│   ├── ks-openapi-spec
│   └── openapi-spec
├── build					// 构建成制品
│   ├── ks-apiserver
│   └── ks-controller-manager
├── cmd						// 命令入口
│   ├── controller-manager
│   └── ks-apiserver
├── config				// 生成crds等配置文件
│   ├── crds
│   ├── gateway
│   ├── ks-core
│   └── watches.yaml
├── docs					// 帮助文档
│   ├── build-multiarch-images.md
│   ├── images
│   ├── powered-by-kubesphere.md
│   └── roadmap.md
├── go.mod
├── go.sum
├── hack					// 快速生操作等脚本
│   ├── boilerplate.go.txt
├── install				// 废弃
│   ├── ingress-controller
│   ├── scripts
│   └── swagger-ui
├── kube
│   ├── pkg
│   └── plugin
├── pkg						// 主要业务逻辑目录
│   ├── api				// k8s api目录
│   ├── apis			// crds定义api
│   ├── apiserver
│   ├── client		// used by code-generator, informer/lister/clientset
│   ├── constants
│   ├── controller	// controllers
│   ├── informers
│   ├── kapis			// KubeSphere specific apis, api path starts with /kapis
│   ├── models
│   ├── server
│   ├── simple
│   ├── test
│   ├── tools.go
│   ├── utils
│   ├── version
│   └── webhook
├── staging
│   ├── README.md
│   ├── publishing
│   └── src
├── test
│   ├── e2e
│   ├── network
│   └── testdata
├── tools
│   ├── cmd
│   ├── lib
│   └── tools.go
└── vendor
    ├── code.cloudfoundry.org
    ├── github.com
    ├── go.etcd.io
    ├── go.mongodb.org
    ├── go.starlark.net
    ├── go.uber.org
    ├── golang.org
    ├── gomodules.xyz
    ├── google.golang.org
    ├── gopkg.in
    ├── gotest.tools
    ├── helm.sh
    ├── honnef.co
    ├── istio.io
    ├── k8s.io
    ├── kubesphere.io
    ├── modules.txt
    └── sigs.k8s.io

```

## 三 本地构建

```shell
$ make test    // takes a really long time
$ make all     // build ks-apiserver/ks-controller-manager 

$ make ks-apiserver
$ go build -o bin/cmd/ks-apiserver cmd/ks-apiserver/apiserver.go


$ 获取配置文件
$ bin/cmd/ks-apiserver  --kubeconfig ~/.kube/config


```

## 四 本地部署最小化

```shell
kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.2.1/kubesphere-installer.yaml
   
kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.2.1/cluster-configuration.yaml
```







## 参考链接

* https://kubesphere.com.cn/forum/d/2956-kubesphere/3
* https://github.com/kubesphere/community/blob/master/developer-guide/development/quickstart.md
* https://github.com/kubesphere/community/tree/master/developer-guide/development