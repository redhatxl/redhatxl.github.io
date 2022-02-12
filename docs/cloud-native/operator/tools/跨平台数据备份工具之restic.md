

# 跨平台数据备份工具之restic详解

# 一 背景

Restic 是一款 GO 语言开发的开源免费且快速、高效和安全的跨平台备份工具。Restic 使用加密技术来保证你的数据安全性和完整性，可以将本地数据加密后传输到指定的存储。

Restic 同样支持增量备份，可随时备份和恢复备份。Restic 支持大多数主流操作系统，比如：Linux、macOS、Windows 以及一些较小众的操作系统 FreeBSD 和 OpenBSD 等。



# 二 restic简介

## 2.1 restic 支持类型

- 本地存储
- SFTP
- REST Server
- Amazon S3
- Minio Server
- OpenStack Swift
- Backblaze B2
- Microsoft Azure Blob Storage
- Google Cloud Storage
- 通过 Rclone 挂载的存储 (比如：Google Drive、OneDrive 等)

## 2.2 **Restic 与 Rclone** 区别

### 2.2.1 相同点

- 两者都是基于命令行的开源文件同步和备份工具。
- 两者都支持将文件备份到本地、远程服务器或对象存储。

### 2.2.2 异同

- Rclone 面向的是文件同步，即保证两端文件的一致，也可以增量备份。
- Restic 面向的是文件备份和加密，文件先加密再传输备份，而且是增量备份，即每次只备份变化的部分。
- Rclone 仓库配置保存在本地，备份的文件会保持原样的同步于存储仓库中。
- Restic 配置信息直接写在仓库，只要有仓库密码，在任何安装了 Restic 的计算机上都可以操作仓库。
- Rclone 不记录文件版本，无法根据某一次备份找回特定时间点上的文件。
- Restic 每次备份都会生成一个快照，记录当前时间点的文件结构，可以找回特定时间点的文件。
- Rclone 可以在配置的多个存储端之间传输文件。

总的来说，Rclone 和 Restic 各有所长，要根据不同的业务需求选择使用。比如：网站数据的增量备份，用 Resitc 就比较合适。而常规文件的远程备份归档，用 Rclone 就很合适。

## 2.3 Restic设计原则

Restic 是一个可以正确进行备份的程序，其设计遵循以下原则： 

* 简单：备份应该是一个顺畅的过程，否则您可能会想跳过它。 Restic 应该易于配置和使用，以便在数据丢失的情况下，您可以直接恢复它。同样，恢复数据不应该很复杂。 
* 快速：用restic备份你的数据应该只受你的网络或硬盘带宽的限制，这样你就可以每天备份你的文件。如果需要太多时间，没有人会进行备份。恢复备份应该只传输要恢复的文件所需的数据，这样这个过程也很快。 
* 可验证：比备份更重要的是恢复，因此restic使您可以轻松验证所有数据是否可以恢复。
*  安全：Restic 使用加密技术来保证您数据的机密性和完整性。假设存储备份数据的位置不是受信任的环境（例如，系统管理员等其他人能够访问您的备份的共享空间）。 Restic 旨在保护您的数据免受此类攻击者的侵害。 
* 高效：随着数据的增长，额外的快照应该只占用实际增量的存储。更重要的是，在将重复数据实际写入存储后端之前，应该对其进行去重，以节省宝贵的备份空间。



## 2.4 相关术语

* *Repository*：备份期间产生的所有数据都以结构化形式发送并存储在存储库中，例如在具有多个子目录的文件系统层次结构中。存储库实现必须能够完成许多操作，例如列出内容。v0.12.0中已支持的存储服务包括：aws s3，minio server，Wasabi， Aliyun OSS， OpenStack Swift，Backlbaze B2，Azure Blob Storage，Google Cloud Storage，rclone*

* *Blob*：Blob 将多个数据字节与识别信息（如数据的 SHA-256 哈希及其长度）,加密的数据块及元数据，其中元数据包括长度，SHA-256 哈希信息。数据块可以存放文件数据（data），也可以存放目录结构数据（tree）。Blob的大小在512KiB到8MiB之间，因此小于512KB的文件不会被拆分。Restic的实现目标是让Blob平均大小为1MiB。

* *Pack*：一个包结合了一个或多个 Blob，例如在单个文件中。Restic中的单个数据文件，包括一个或多个Blob，一旦创建不再修改。

  一般只创建不删除，仅prune操作会删除不再被引用的数据。

* *Snapshot*：快照代表在某个时间点已备份的文件或目录的状态。这里的状态是指内容和元数据，如文件或目录及其内容的名称和修改时间。

* *Storage ID*：Pack文件的SHA256哈希值，通过这个ID可以在仓库中加载需要的数据文件。Restic将这个ID作为Pack的文件名，也就是文件的SHA256哈希值。Pack文件名即哈希值的设计也可以方便的检验数据文件是否被改动过。

# 三 安装restic

## 3.1 yum安装

```shell
yum install yum-plugin-copr
yum copr enable copart/restic
yum install restic

```

## 3.2 docker安装

```
docker pull restic/restic
```

更多信息可参考：https://github.com/Lobaro/restic-backup-docker

## 3.3 源码安装

```shell
$ git clone https://github.com/restic/restic

$ cd restic

$ go run build.go
```

## 3.4 配置自动补全

```
$ sudo ./restic generate --bash-completion /etc/bash_completion.d/restic
```

# 四 实战

将保存备份的地方称为“存储库”。本章解释了如何创建(“ init”)这样的存储库。存储库可以存储在本地，也可以存储在远程服务器或服务器上。

## 4.1 sftp主机间备份

### 4.1.1 主机间免密钥互信

从主机A备份数据到主机B，需要主机A到主机B免密钥和互信

```shell
ssh-keygen -t rsa
ssh-copy-id -i /root/.ssh/id_rsa.pub root@106.53.117.41
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210902131720.png)

### 4.1.2 在服务器A创建备份

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210902132127.png)

初始化备份，/data为 B服务器目录。

查看B服务器

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210902132357.png)

### 4.1.3 备份操作

* 执行数据备份

```shell
restic -r sftp:root@106.53.117.41:/data backup ./
```



* 查看备份

```shell
restic -r sftp:root@106.53.117.41:/data snapshots
```

* 查看备份内容

```
restic -r sftp:root@106.53.117.41:/data ls 875a2a32
```

* 恢复快照

```shell
restic -r sftp:root@106.53.117.41:/data restore 875a2a32 -t ./
restic -r sftp:root@106.53.117.41:/data restore 875a2a32 --target ./
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210902140919.png)

* 删除备份

```shell
restic -r sftp:root@106.53.117.41:/data forget 875a2a32
```

### 4.1.4 免密码

上面备份的时候，都需要输入密码，肯定不适合脚本自动备份，所以我们还需要使用`--password-file`参数来达到自动读取密码的步骤。

```
#先将密码，比如moerats保存在/root/resticpasswd文本中
echo 'xxzx@789' > /root/resticpasswd
#然后在备份命令中加--password-file参数来读取文本中的密码，这里以sftp为例
restic -r sftp:root@106.53.117.41:/data --verbose backup ./ --password-file /root/resticpasswd
```

## 4.2 对象存储备份

支持基于s3协议的后端对象存储，例如minio或者腾讯/阿里对象存储

### 4.2.1 阿里云对象存储

```shell
$ export AWS_ACCESS_KEY_ID=<YOUR-OSS-ACCESS-KEY-ID>
$ export AWS_SECRET_ACCESS_KEY=<YOUR-OSS-SECRET-ACCESS-KEY>

$ ./restic -o s3.bucket-lookup=dns -o s3.region=<OSS-REGION> -r s3:https://<OSS-ENDPOINT>/<OSS-BUCKET-NAME> init
$ restic -o s3.bucket-lookup=dns -o s3.region=oss-eu-west-1 -r s3:https://oss-eu-west-1.aliyuncs.com/bucketname init

restic -o s3.bucket-lookup=dns -o oss-cn-beijing.aliyuncs.com -r s3:https://xueltestoss.oss-cn-beijing.aliyuncs.com init

```

* 创建repository

```shell
export AWS_ACCESS_KEY_ID=LTAI4Fd9eKBo1daUHS7RdZa9
export AWS_SECRET_ACCESS_KEY=XvHrpwKRuvNzaSoloExRU3fYJt3wb7
restic -o s3.bucket-lookup=dns -o s3.region=oss-cn-beijing.aliyuncs.com -r s3:https://xueltestoss.oss-cn-beijing.aliyuncs.com/xueltestoss init
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210902143250.png)

对象存储上文件

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210902143310.png)

* 免密钥

```shell
#先将密码，比如moerats保存在/root/resticpasswd文本中
echo 'xxzx@789' > /root/resticpasswd
#然后在备份命令中加--password-file参数来读取文本中的密码，这里以sftp为例
```

* 执行备份

```shell
restic -r s3:https://oss-cn-beijing.aliyuncs.com/xueltestoss --password-file /root/resticpasswd backup /data/
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210902145003.png)

其他恢复操作基本上和sftp的一致。

# 其他

restic 是一个很不错的数据备份方案，rclone是一个不错的数据同步方案，以及minio作为数据存储，集成在一起真的很不错。

# 参考文章

* https://restic.readthedocs.io/en/v0.12.0/design.html
* https://github.com/restic/restic
* https://restic.net/