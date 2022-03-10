## OCI镜像制作工具之skopeo

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220309231302.png)

## 前言



## 一 slopeo简介

### 1.1 简介

skopeo 是一个命令行工具，用于对容器镜像和镜像库执行各种操作，支持使用 OCI 镜像与原始的 Docker v2 镜像。可对容器镜像和容器存储进行操作。 在没有dockerd的环境下，使用 **skopeo** 操作镜像是非常方便的。Skopeo 能够在容器注册表上检查存储库并获取图片层。Inspect 命令获取存储库的清单，它能够向您显示关于整个存储库或标记的类似 docker inspect 的 json 输出。与 docker inspect 不同，这个工具可以帮助您在拉取存储库或标记之前(使用磁盘空间)收集有用的信息。Inspect 命令可以显示给定存储库可用的标记、图像的标签、图像的创建日期和操作系统等等。

### 1.2 特性

skopeo 使用 API V2 注册表，例如 Docker 注册表、Atomic 注册表、私有注册表、本地目录和本地 OCI 布局目录。skopeo
不需要运行守护进程，它可以执行的操作包括：

- 通过各种存储机制复制镜像，例如，可以在不需要特权的情况下将镜像从一个注册表复制到另一个注册表
- 检测远程镜像并查看其属性，包括其图层，无需将镜像拉到本地
- 从镜像库中删除镜像
- 当存储库需要时，skopeo 可以传递适当的凭据和证书进行身份验证

## 二 安装

* RHEL/CentOS ≤ 7.x

```
sudo yum -y install skopeo
```

* macOS

```
brew install skopeo
```

详细可参考：https://github.com/containers/skopeo/blob/main/install.md

## 三 使用

```shell
$ skopeo -h
Various operations with container images and container image registries

Usage:
  skopeo [flags]
  skopeo [command]

Available Commands:
  copy                                          Copy an IMAGE-NAME from one location to another
  delete                                        Delete image IMAGE-NAME
  help                                          Help about any command
  inspect                                       Inspect image IMAGE-NAME
  list-tags                                     List tags in the transport/repository specified by the REPOSITORY-NAME
  login                                         Login to a container registry
  logout                                        Logout of a container registry
  manifest-digest                               Compute a manifest digest of a file
  standalone-sign                               Create a signature using local files
  standalone-verify                             Verify a signature using local files
  sync                                          Synchronize one or more images from one location to another

Flags:
      --command-timeout duration   timeout for the command execution
      --debug                      enable debug output
  -h, --help                       help for skopeo
      --insecure-policy            run the tool without any policy check
      --override-arch ARCH         use ARCH instead of the architecture of the machine for choosing images
      --override-os OS             use OS instead of the running OS for choosing images
      --override-variant VARIANT   use VARIANT instead of the running architecture variant for choosing images
      --policy string              Path to a trust policy file
      --registries.d DIR           use registry configuration files in DIR (e.g. for container signature storage)
      --tmpdir string              directory used to store temporary files
  -v, --version                    Version for Skopeo

Use "skopeo [command] --help" for more information about a command.
```

### 3.1 检查存储库

* 显示镜像属性
  * 远程仓库

`docker://`: 是使用 Docker Registry HTTP API V2 进行连接远端

`docker.io`: 远程仓库

`fedora:latest`: 镜像名称及tag

```shell
$ skopeo inspect docker://docker.io/fedora:latest
```

利用jq查看指定的信息。

```
$ skopeo inspect docker://registry.fedoraproject.org/fedora:latest | jq .Digest
```

* 查看本地镜像

```shell
$ skopeo inspect docker-daemon:mysql:8.0.19
{
    "Name": "docker.io/library/mysql",
    "Digest": "sha256:8084d869573a2e1be9d49d1b8dea49b54fc43301a3ead4b8124eedb809cf3c3f",
    "RepoTags": [],
    "Created": "2020-04-23T04:14:55.648457026Z",
    "DockerVersion": "18.09.7",
    "Labels": null,
    "Architecture": "amd64",
    "Os": "linux",
    "Layers": [
        "sha256:c2adabaecedbda0af72b153c6499a0555f3a769d52370469d8f6bd6328af9b13",
        "sha256:49003fe88142d189c33a6e52b97081c8d62b67e0d04e9e7421ca6a2495b0d766",
        "sha256:8d3b3830445d713c580008a547de7398ab721a083fda915526ce2cc557c4f40a",
        "sha256:49baacc63c3bb5a7ab5077ef6458d6e246710195d7794cb704940f3a40495c23",
        "sha256:24bd91e7be374167e51e37faa31e7ce143921afb0b968415c749625319386fdb",
        "sha256:d84f8cf1dc236ea2ddbc0393b0c33099bedb20fffbe68bdf170d2df161a1ca16",
        "sha256:ace74cb61ec0a3fd2847dc3a0b8b0cad349c95ce02732982e258fc1a337d073f",
        "sha256:ba6f785586f5ff477e87528a5d2c8d0215e22f5b9ac3a8f4b19a033cad7aa832",
        "sha256:c861c8d6e01539cc5b27fe773aca63958bc361391f6a980ece5322fd8dbca296",
        "sha256:1ac7b6bb690abc8073c2193635afb06b8494d2d0024dc08310b296975f92bb61",
        "sha256:b8e742ca24b882430bf4a6075aa97a8c1e8809f6e59d462cba05bec49d817ff3",
        "sha256:7fc1a766ce1413db3d5fceb7256c1e54639386673c24eadaf69b429b07eb9223"
    ],
    "Env": [
        "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
        "GOSU_VERSION=1.12",
        "MYSQL_MAJOR=8.0",
        "MYSQL_VERSION=8.0.19-1debian10"
    ]
}
```

### 3.2 拷贝镜像

在不使用 docker 的情况下从远端下载镜像

```shell
# 下载远程镜像到/tmp/下，命名为nginx.tar
skopeo --insecure-policy copy docker://nginx:1.17.6 docker-archive:/tmp/nginx.tar

# 将本地tar镜像文件导入到docker中
skopeo copy docker-archive:/tmp/nginx.tar docker-daemon:nginx:latest
```

### 3.3 删除镜像

```
skopeo delete docker://localhost:5000/nginx:latest
```

### 3.4 注册表认证

认证文件默认存放在` $HOME/.docker/config.json`

文件内容

```
{
	"auths": {
		"myregistrydomain.com:5000": {
			"auth": "dGVzdHVzZXI6dGVzdHBhc3N3b3Jk",
			"email": "stuf@ex.cm"
		}
	}
}
```

## 参考链接

* https://www.redhat.com/zh/blog/skopeo-copy-rescue
* https://github.com/containers/skopeo
* https://lework.github.io/2020/04/13/skopeo/