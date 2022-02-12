# K8s日志查看利器之kubetail

# 一 背景

Kubetail 是一个小型 bash 脚本，其能够将来自于多个 pod 的日志聚合到同一数据流中。Kubetail 的初始版本不提供过滤或高亮功能，但其目前已经在 GitHub 上添加了一个分支，该分支支持使用 multitail 工具构建日志并对日志着色。

对于日常如果不借助日志系统，对于一个deployment多个副本，查看日志如果分多个终端查看单独对应一个pod非得的麻烦而且不便于观察，此刻kubetail可以将多个pod日志进行聚合，非常方便日常查看。

# 二 安装部署

## 2.1 macos安装

```shell
brew tap johanhaleby/kubetail && brew install kubetail
```

## 2.2 Linux安装

```shell
wget https://github.com/johanhaleby/kubetail/archive/1.6.12.tar.gz
tar -zxvf 1.6.12.tar.gz
cd kubetail-1.6.12/
cp kubetail /usr/bin/

# 配置命令补全
cp completion/kubetail.bash /etc/bash_completion.d/
```

# 三 基本用法

* 语法

```shell
$ kubetail <search term> [-h] [-c] [-n] [-t] [-l] [-d] [-p] [-s] [-b] [-k] [-v] [-r] [-i] 
```

* 参数解释：

```shell
-c：指定多容器 Pod 中的容器名称
-t：指定 Kubeconfig 文件中的 Context
-l：标签过滤器，使用 -l 参数之后，会忽略 Pod 名称
-n：指定命名空间
-s：指定返回一个相对时间之后的日志，例如 5s，2m 或者 3h，缺省是 10s
-b：是否使用 line-buffered，缺省为 false
-k：指定输出内容的具体着色部分，pod：只给 pod 名称上色，line：整行上色（缺省），false：不上色
```

* 实例：

```shell
$ kubetail my-pod-v1
$ kubetail my-pod-v1 -c my-container
$ kubetail my-pod-v1 -t int1-context -c my-container
$ kubetail '(service|consumer|thing)' -e regex
$ kubetail -l service=my-service
$ kubetail --selector service=my-service --since 10m
$ kubetail --tail 1
```

# 四 测试

```shell
 xuel@kaliarchmacbookpro  ~  kubectl get po -n default |grep smart
smart-bc8c658cf-6ffpx                2/2       Running   3          2d
smart-5f99d9bdcc-lqr8g                  1/1       Running   0          84d
smart-765dbf6f-7fvjn                  2/2       Running   0          84d
smart-765dbf6f-wjz42                  2/2       Running   0          84d

# ns写在最后
 xuel@kaliarchmacbookpro  ~  kubetail smart-celery-765dbf6f -n default
Will tail 4 logs...
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20201220204026.png)

# 五 其他

kubetail可以非常方便的查看多个容器日志，其本身为shell脚本，安装非常方便，在没有使用日志系统查看日志的时候，登录集群使用kubetail一个deployment多个pod副本非常的方便。	

# 六 参考链接

* https://github.com/johanhaleby/kubetail



