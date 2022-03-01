# 镜像安全扫描之Trivy

# 一 概述

## 1.1 Trivy简介

Trivy 是一种适用于 CI 的简单而全面的容器漏洞扫描程序。软件漏洞是指软件或操作系统中存在的故障、缺陷或弱点。Trivy 检测操作系统包（Alpine、RHEL、CentOS 等）和应用程序依赖（Bundler、Composer、npm、yarn 等）的漏洞。Trivy 很容易使用，只要安装二进制文件，就可以扫描了。扫描只需指定容器的镜像名称。与其他镜像扫描工具相比，例如 Clair，Anchore Engine，Quay 相比，Trivy 在准确性、方便性和对 CI 的支持等方面都有着明显的优势。

## 1.2 特性

1. 检测面很全，能检测全面的漏洞，操作系统软件包（Alpine、Red Hat Universal Base Image、Red Hat Enterprise Linux、CentOS、Oracle Linux、Debian、Ubuntu、Amazon Linux、openSUSE Leap、SUSE Enterprise Linux、Photon OS 和 Distrioless）、应用程序依赖项（Bundler、Composer、Pipenv、Poetry、npm、yarn 和 Cargo）；
2. 使用简单，仅仅只需要指定镜像名称；
3. 扫描快且无状态，第一次扫描将在 10 秒内完成（取决于您的网络）。随后的扫描将在一秒钟内完成。与其他扫描器在第一次运行时需要很长时间（大约 10 分钟）来获取漏洞信息，并鼓励您维护持久的漏洞数据库不同，Trivy 是无状态的，不需要维护或准备；
4. 易于安装，安装方式：

```shell
   1. apt-get install
   2. yum install
   3. brew install
```
无需安装数据库、库等先决条件（例外情况是需要安装 rpm 以扫描基于 RHEL/CentOS 的图像）。

# 二 安装部署

## 2.1 Linux安装

```shell
$ sudo vim /etc/yum.repos.d/trivy.repo
[trivy]
name=Trivy repository
baseurl=https://knqyf263.github.io/trivy-repo/rpm/releases/$releasever/$basearch/
gpgcheck=0
enabled=1
$ sudo yum -y update
$ sudo yum -y install trivy
```

## 2.2 Mac OS安装

```shell
$ brew install knqyf263/trivy/trivy
```

## 2.3 使用方式

```shell
当然，trivy也支持docker快速启动

docker run --rm -v [YOUR_CACHE_DIR]:/root/.cache/ knqyf263/trivy [YOUR_IMAGE_NAME]

需要将YOUR_CACHE_DIR替换成你的缓存目录，将之前缓存的漏洞库映射到容器中，不然启动会很慢



如果想通过这种方式扫描本机的容器镜像，需要将docker.sock文件映射到容器内，比如

docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \

    -v $HOME/.cache/:/root/.cache/ knqyf263/trivy python:3.4-alpine



trivy使用简单示例如下：

扫描镜像：

trivy [image_name]



扫描通过save方式打包的镜像：

trivy --input [image_name.tar]



将扫描结果存为json文件：

trivy -f json -o results.json [image_name]



通过设置漏洞级别过滤漏洞：

trivy --severity HIGH,CRITICAL [image_name]



通过设置漏洞类型过滤漏洞：

trivy --vuln-type os [image_name]

漏洞类型分为os和library



只更新某个系统类型的漏洞库：

trivy --only-update alpine,debia [image_name]



指定退出代码：

trivy --exit-code 1 [image_name]

指定退出代码，主要是为后续判断提供可操作性，主要是在CI中



由于trivy有缓存，所以在扫描镜像的latest版本的时候，会出现异常，需要清楚缓存操作：

trivy --clear-cache [image_name]



如果需要重建本地漏洞数据库，或清除所有缓存，可以通过trivy --reset
```

# 三 集成到CI

Trivy 有对 CI 友好的特点，并且官方也以这种方式使用它，想要集成 CI 只需要一段简单的 Yml 配置文件即可，如果发现漏洞，测试将失败。如果不希望测试失败，请指定--exit code 0。由于在自动化场景（如 CI/CD）中，您只对最终结果感兴趣，而不是对完整的报告感兴趣，因此请使用--light 标志对此场景进行优化，以获得快速的结果。

集成 GitLab CI 的 Yml 配置可以参考：

https://github.com/aquasecurity/trivy#gitlab-ci

# 四 注意事项

- 国内拉取漏洞数据库慢。
- 同一台服务器，多个镜像扫描的时候不可并行执行。
- 可以使用--light 使用轻量级数据库来优化执行扫描的效率。

# 五 反思

trivy 不仅可以集成在 CI 过程中，及时发现镜像漏洞，而且可以于自建 harbor 进行集成，定时对已经上传的镜像进行扫描。

# 参考链接

* https://github.com/aquasecurity/trivy

* https://blog.csdn.net/qq_31977125/article/details/105639085
* https://mp.weixin.qq.com/s/n9T2kcuAMOL5Dj-_d7IH6g
* https://my.oschina.net/u/3495789/blog/4346263