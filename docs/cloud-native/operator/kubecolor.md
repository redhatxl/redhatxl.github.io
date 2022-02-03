## Kubecolor 色彩化输出Kubernetes

# 一 背景



`kubectl`命令是k8s的`CLI`工具，如果你是维护K8s集群的管理员或者是开发可在Kubernetes上运行的应用程序的开发人员，那几乎每天都会使用kubectl，但是尽管kubectl已经很好，它依旧有些地方让人十分的头疼。比如`缺少颜色`，kubectl的输出`有时不容易阅读，由于kubectl有时会输出很长的内容，因此很难找到所需的内容`。因此如果有个能高亮颜色显示输出的工具，看起来就相对的更加直观了，所以`kubecolor`来了。

# 二 安装

## 2.1 mac安装

```shell
$ brew install dty1er/tap/kubecolor
```

我这边终端使用的是`iterm2`和`oh-my-zsh`，因此这里直接在`vim ./.zshrc`修改就可以了,比如我的文件内容

```shell
# kubectl get resource
alias kubectl="kubecolor"
alias k="kubecolor"
alias kn="kubectl get nodes -o wide"
alias kp="kubectl get pods -o wide"
alias kd="kubectl get deployment -o wide"
alias ks="kubectl get svc -o wide"
# kubectl describe resources
alias kdp="kubectl describe pod"
alias kdd="kubectl describe deployment"
alias kds="kubectl describe service"
alias kdn="kubectl describe node"
```

## 2.2 go 安装

``` 
$ go get -u github.com/dty1er/kubecolor/cmd/kubecolor
```

# 三 使用

* 查看k8s资源

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211002202409.png)

* describe

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211002202457.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211003194620.png)