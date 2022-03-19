# K8s快捷操作工具之K9s

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220319151804.png)

## 一 前言

通常我们在查看K8s集群的时候，如果没有web界面，需要利用kubectl命令来查看，输入很长的命令，单不一定一次性对，而且重复命令需要再次输入，为了解决该问题，K9s可以帮我们快速实现查看K8s各类资源，编辑，修改，更新等操作。

## 二 K9s简介

### 2.1 K9s简介

K9s是一个基于终端的UI，用于与Kubernetes集群交互。这个项目的目的是使导航、观察和管理已部署的应用程序更容易。K9s不断地监视Kubernetes的变化，并提供后续命令来与观察到的资源交互。

### 2.2 特性

- **信息触手可及**

- - 跟踪 `Kubernetes` 集群中运行的资源的实时活动
  - 处理 `Kubernetes` 标准资源和自定义资源定义

- **集群指标**

- - 跟踪与 `Pod`，容器和节点等资源关联的实时指标

- **高级特性**

- - 提供标准的集群管理命令，例如日志，扩展，端口转发，重启
  - 定义自己的命令快捷方式，以通过命令别名和热键快速导航
  - 支持插件扩展 `k9s` 来创建属于自己的集群操作管理命令
  - 强大的过滤模式，允许用户向下钻取并查看与工作负载相关的资源

- **外观可定制**

- - 通过 `K9s` 皮肤定义自己的外观
  - 自定义/安排要按资源显示的列

- **Pulses-集群事务状态的顶级仪表板**

## 三 安装部署

- MacOS

  ```shell
   # Via Homebrew
   brew install derailed/k9s/k9s
   # Via MacPort
   sudo port install k9s
  ```

- Linux

  ```shell
   # Via LinuxBrew
   brew install derailed/k9s/k9s
   # Via PacMan
   pacman -S k9s
  ```

- Windows

  ```shell l
  # Via scoop
  scoop install k9s
  # Via chocolatey
  choco install k9s
  ```

## 四 使用

### 4.1 命令

```shell
$ k9s -h
K9s is a CLI to view and manage your Kubernetes clusters.

Usage:
  k9s [flags]
  k9s [command]

Available Commands:
  completion  generate the autocompletion script for the specified shell
  help        Help about any command
  info        Print configuration info
  version     Print version/build info

Flags:
  -A, --all-namespaces                 Launch K9s in all namespaces
      --as string                      Username to impersonate for the operation
      --as-group stringArray           Group to impersonate for the operation
      --certificate-authority string   Path to a cert file for the certificate authority
      --client-certificate string      Path to a client certificate file for TLS
      --client-key string              Path to a client key file for TLS
      --cluster string                 The name of the kubeconfig cluster to use
  -c, --command string                 Overrides the default resource to load when the application launches
      --context string                 The name of the kubeconfig context to use
      --crumbsless                     Turn K9s crumbs off
      --headless                       Turn K9s header off
  -h, --help                           help for k9s
      --insecure-skip-tls-verify       If true, the server's caCertFile will not be checked for validity
      --kubeconfig string              Path to the kubeconfig file to use for CLI requests
      --logFile string                 Specify the log file (default "/var/folders/wn/367g1v9n1bv0sg1k8qldzym80000gn/T/k9s-xuel.log")
  -l, --logLevel string                Specify a log level (info, warn, debug, trace, error) (default "info")
      --logoless                       Turn K9s logo off
  -n, --namespace string               If present, the namespace scope for this CLI request
      --readonly                       Sets readOnly mode by overriding readOnly configuration setting
  -r, --refresh int                    Specify the default refresh rate as an integer (sec) (default 2)
      --request-timeout string         The length of time to wait before giving up on a single server request
      --screen-dump-dir string         Sets a path to a dir for a screen dumps
      --token string                   Bearer token for authentication to the API server
      --user string                    The name of the kubeconfig user to use
      --write                          Sets write mode by overriding the readOnly configuration setting

Use "k9s [command] --help" for more information about a command.
```

可以通过`k9s info`查看k9s的配置文件信息

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220319152407.png)

### 4.2 使用

直接输入k9s则直接进入需要查看的K8s集群内部。



![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220319152523.png)

* 对Pod资源进行查看：

`d`：describe pod

`l`：logs pod

`a`：attach pod

`e`：edit pod

* 对其他资源进行查看

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220319152706.png)

`:svc`：查看svc资源

`:deploy`：查看deploy资源

`:cm`：查看configmap资源

`:rb`：查看rbac资源

## 参考链接

* https://github.com/derailed/k9s
* https://k9scli.io/

