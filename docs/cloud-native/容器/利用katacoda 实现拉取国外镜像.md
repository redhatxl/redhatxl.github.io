# 利用katacoda实现拉取国外镜像

## 一 背景

安装 [kubernetes](https://so.csdn.net/so/search?from=pc_blog_highlight&q=kubernetes) 的时候，我们需要用到 `gcr.io/google_containers` 下面的一些镜像，在国内是不能直接下载的。如果用 Self Host 方式安装，Master 上的组件除开 Kubelet 之外都用容器运行，甚至 CNI 插件也是容器运行。比如 Flannel，在 `quay.io/coreos` 下面，在国内下载非常慢。但是我们可以把这些镜像同步到我们的 Docker Hub 仓库里，再配个 Docker Hub 加速器，这样下载镜像就很快了。

## 二 原理

Katacoda 是一个在线学习平台，在 Web 上提供学习需要的服务器终端，里面包含学习所需的环境，我们可以利用 `Docker` 课程的终端来同步，因为里面有 `Docker` 环境，可以执行 docker login、docker pull、docker tag、docker push 等命令来实现同步镜像。

但是手工去执行命令很麻烦，如果要同步的镜像和 Tag 比较多，手工操作那就是浪费生命。我们可以利用程序代替手工操作，不过 Katacoda 为了安全起见，不允许执行外来的二进制程序，但是可以 Shell 脚本，我写好了脚本，大家只需要粘贴进去根据自己需要稍稍修改下，然后运行就可以了。

## 三 操作步骤

### 3.1 登陆katacoda

点击 [这里](https://www.katacoda.com/courses/docker/deploying-first-container) 进入 Docker 课程。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211112110026.png)

点击 `START SCENARIO` 或 终端右上角全屏按钮将终端放大。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211112110005.png)

### 3.2 登陆dockerhub

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211112110722.png)

利用下载

```shell
k8s.gcr.io/metrics-server/metrics-server
```



### 3.3 下载镜像

```shell
docker pull k8s.gcr.io/metrics-server/metrics-server:v0.5.0
docker tag k8s.gcr.io/metrics-server/metrics-server:v0.5.0 1832990/metrics-server:v0.5.0
docker push 1832990/metrics-server:v0.5.0
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211112110835.png)

之后在国内就可以使用我们自己的镜像了

```shell
1832990/metrics-server:v0.5.0
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211112111043.png)

## 四 反思

可以自定义脚本来更快实现

```shell
#!/bin/bash
sourceRegistry=""
myRegistry=""

image=""

# docker login

# docker pull 

# docker tag

# docker push
```

## 参考链接

* https://katacoda.com/
* https://blog.csdn.net/easylife206/article/details/98818707