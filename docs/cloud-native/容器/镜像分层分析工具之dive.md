## 镜像分层分析工具之dive

## 前言

我们知道Docker的镜像是由一层一层联合文件系统构成，在Dockerfile中，每写一层就对应镜像中的一层，那么我们如果对已经制作好的镜像进行分层的查看和审核，以及后期的优化呢，dive就是一款镜像分层分析工具，可以帮助我们去实现。

## 一 dive简介

### 1.1 简介

用于探索 Docker 映像、图层内容和发现缩小 Docker/OCI 映像大小的方法的工具。

### 1.2 特性

* 显示按层分解的 Docker 图像内容

当您选择左侧的图层时，您会看到该图层的内容以及右侧的所有先前图层。此外，您可以使用箭头键全面探索文件树。

* 指出每一层的变化

已更改、修改、添加或删除的文件在文件树中显示。这可以调整以显示特定层的更改，或聚合到该层的更改。

* 估计“图像效率”

左下方的面板显示了基本的图层信息和一个实验性的指标，来猜测你的图片包含了多少浪费的空间。这可能是由于跨层复制文件，跨层移动文件，或者没有完全删除文件。提供了百分比“得分”和总浪费的文件空间

* 快速构建/分析周期

您可以使用一个命令构建 Docker 映像并立即进行分析：dive build -t some-tag。 您只需要用相同的潜水构建命令替换您的 docker build 命令。

* CI 集成

根据图像效率和浪费的空间分析图像并获得通过/失败结果。调用任何有效的潜水命令时，只需在环境中设置 CI=true。

## 二 安装

* RHEL/CentOS

```shell
curl -OL https://github.com/wagoodman/dive/releases/download/v0.9.2/dive_0.9.2_linux_amd64.rpm
rpm -i dive_0.9.2_linux_amd64.rpm
```

* Mac

```shell
brew install dive
```

## 三 使用

* KeyBindings

| Key Binding                               | Description                         |
| :---------------------------------------- | :---------------------------------- |
| <kbd>Ctrl + C</kbd>                       | 退出                                |
| <kbd>Tab</kbd> or <kbd>Ctrl + Space</kbd> | 在图层和文件树视图之间切换          |
| <kbd>Ctrl + F</kbd>                       | 过滤文件                            |
| <kbd>Ctrl + A</kbd>                       | 图层视图：查看聚合图像修改          |
| <kbd>Ctrl + L</kbd>                       | 图层视图：查看当前图层修改          |
| <kbd>Space</kbd>                          | Filetree视图：折叠/取消折叠目录     |
| <kbd>Ctrl + A</kbd>                       | Filetree视图： 显示/隐藏添加的文件  |
| <kbd>Ctrl + R</kbd>                       | Filetree视图：显示/隐藏已删除的文件 |
| <kbd>Ctrl + M</kbd>                       | Filetree视图：显示/隐藏已修改的文件 |
| <kbd>Ctrl + U</kbd>                       | Filetree视图：显示/隐藏未修改的文件 |
| <kbd>PageUp</kbd>                         | Filetree视图：向上滚动页面          |
| <kbd>PageDown</kbd>                       | Filetree视图：向下滚动页面          |

* 分析mysql镜像

```shell
dive mysql:8.0.19
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220309230421.png)

* 执行CI扫描

```shell
dive mysql:8.0.19 --ci
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220309230637.png)

## 其他

在我们CI中可以集成dive，对每次构建的镜像进行检测，及时发现镜像中的异常和优化点，针对性的优化镜像。

## 参考链接

* https://github.com/wagoodman/dive