# IaC扫描工具之Terrascan

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220111110242.png)

# 一 背景

对于基础设施，例如dockerfile/kubernetes资源清单，还有terraform.tf 文件，需要进行检测扫描。

# 二 Terrasacn简介

## 2.1 Terrascan简介

Terrascan 是基础架构即代码的静态代码分析器。它可以以多种不同的方式安装和运行，最常用于自动化管道中，以在配置不安全的基础架构之前识别策略违规行为。

## 2.2 特性

Terrascan 是基础架构即代码的静态代码分析器。 Terrascan 允许您：

* 无缝地扫描基础设施作为错误配置的代码。
* 监视有准备的云基础设施，以防止引入姿态漂移的配置更改，并支持恢复到安全姿态。
* 检测安全漏洞和违规行为。
* 在提供云本地基础设施之前降低风险。
* 提供本地运行或与 CI CD 集成的灵活性。

# 三 安装

Terrascan 是一个可移植的可执行文件，不需要安装，也可以作为 Docker Hub 中的容器镜像使用。您可以根据自己的喜好以两种不同的方法使用 Terrascan：

## 3.1 macOS & Linux

```shell

$ curl -L "$(curl -s https://api.github.com/repos/accurics/terrascan/releases/latest | grep -o -E "https://.+?_Darwin_x86_64.tar.gz")" > terrascan.tar.gz
$ tar -xf terrascan.tar.gz terrascan && rm terrascan.tar.gz
$ install terrascan /usr/local/bin && rm terrascan
$ terrascan

```

## 3.2 Docker

```shell
$ docker run --rm accurics/terrascan version

$ alias terrascan="docker run --rm -it -v "$(pwd):/iac" -w /iac accurics/terrascan"

```

# 四 使用

## 4.1 命令行模式

```shelll
$ terrascan
Terrascan

Detect compliance and security violations across Infrastructure as Code to mitigate risk before provisioning cloud native infrastructure.
For more information, please visit https://docs.accurics.com

Usage:
  terrascan [command]

Available Commands:
  help        Provides usage info about any command
  init        Initialize Terrascan
  scan        Start scan to detect compliance and security violations across Infrastructure as Code.
  server      Run Terrascan as an API server
  version     Shows the Terrascan version you are currently using.

Flags:
  -c, --config-path string   config file path
  -h, --help                 help for terrascan
  -l, --log-level string     log level (debug, info, warn, error, panic, fatal) (default "info")
  -x, --log-type string      log output type (console, json) (default "console")
  -o, --output string        output type (human, json, yaml, xml) (default "human")

Use "terrascan [command] --help" for more information about a command.
```

制定特定provider扫描

```shell
# 扫描aws
$ terrascan scan -t aws
# 扫描k8s
$ terrascan scan -i k8s
# 远程扫描
$ terrascan scan -t aws -r git -u git@github.com:accurics/KaiMonkey.git//terraform/aws
# 扫描helm chart
$ terrascan scan -i helm
# 扫描dockerfile
$ terrascan scan -i docker

```

## 4.2 服务端模式

服务器模式将执行 Terrascan 的 API 服务器。当使用 Terrascan 在软件开发管道的多个部分实施一套统一的策略和配置时，这是非常有用的。它还简化了与 Terrascan 的程序交互。默认情况下，http 服务器监听端口9010并支持以下路由:

* 扫描IaC File

```shell
POST - /v1/{iac}/{iacVersion}/{cloud}/local/file/scan

```

POST Parameter: `file` - Content of the file to be scanned

```shell
Example: curl -i -F "file=@aws_cloudfront_distribution.tf" localhost:9010/v1/terraform/v14/aws/local/file/scan
```

* 运行Terrascan为服务端模式

```
$ terrascan server
$ docker run --rm --name terrascan -p 9010:9010 accurics/terrascan

```

* 请求示例子

```shell
$ curl -i -F "file=@aws_cloudfront_distribution.tf" localhost:9010/v1/terraform/v14/aws/local/file/scan
HTTP/1.1 100 Continue

HTTP/1.1 200 OK
Date: Sun, 16 Aug 2020 02:45:35 GMT
Content-Type: text/plain; charset=utf-8
Transfer-Encoding: chunked

{
  "results": {
    "violations": [
      {
        "rule_name": "cloudfrontNoGeoRestriction",
        "description": "Ensure that geo restriction is enabled for your Amazon CloudFront CDN distribution to whitelist or blacklist a country in order to allow or restrict users in specific locations from accessing web application content.",
        "rule_id": "AWS.CloudFront.Network Security.Low.0568",
        "severity": "LOW",
        "category": "Network Security",
        "resource_name": "s3-distribution-TLS-v1",
        "resource_type": "aws_cloudfront_distribution",
        "file": "terrascan-492583054.tf",
        "line": 7
      },
      {
        "rule_name": "cloudfrontNoHTTPSTraffic",
        "description": "Use encrypted connection between CloudFront and origin server",
        "rule_id": "AWS.CloudFront.EncryptionandKeyManagement.High.0407",
        "severity": "HIGH",
        "category": "Encryption and Key Management",
        "resource_name": "s3-distribution-TLS-v1",
        "resource_type": "aws_cloudfront_distribution",
        "file": "terrascan-492583054.tf",
        "line": 7
      },
      {
        "rule_name": "cloudfrontNoHTTPSTraffic",
        "description": "Use encrypted connection between CloudFront and origin server",
        "rule_id": "AWS.CloudFront.EncryptionandKeyManagement.High.0407",
        "severity": "HIGH",
        "category": "Encryption and Key Management",
        "resource_name": "s3-distribution-TLS-v1",
        "resource_type": "aws_cloudfront_distribution",
        "file": "terrascan-492583054.tf",
        "line": 7
      },
      {
        "rule_name": "cloudfrontNoLogging",
        "description": "Ensure that your AWS Cloudfront distributions have the Logging feature enabled in order to track all viewer requests for the content delivered through the Content Delivery Network (CDN).",
        "rule_id": "AWS.CloudFront.Logging.Medium.0567",
        "severity": "MEDIUM",
        "category": "Logging",
        "resource_name": "s3-distribution-TLS-v1",
        "resource_type": "aws_cloudfront_distribution",
        "file": "terrascan-492583054.tf",
        "line": 7
      },
      {
        "rule_name": "cloudfrontNoSecureCiphers",
        "description": "Secure ciphers are not used in CloudFront distribution",
        "rule_id": "AWS.CloudFront.EncryptionandKeyManagement.High.0408",
        "severity": "HIGH",
        "category": "Encryption and Key Management",
        "resource_name": "s3-distribution-TLS-v1",
        "resource_type": "aws_cloudfront_distribution",
        "file": "terrascan-492583054.tf",
        "line": 7
      }
    ],
    "count": {
      "low": 1,
      "medium": 1,
      "high": 3,
      "total": 5
    }
  }
}

```







# 五 CI/CD集成

## 5.1 Gitlab CI

GitLab CI 可以使用 Docker 镜像作为管道的一部分。我们可以利用此功能并将 Terrascan 的 docker 映像用作管道的一部分，以将基础设施作为代码进行扫描。

为此，您可以更新您的。gitlab ci。yml文件使用带有[“bin/sh”、“-c”]入口点的“accurics/terrascan:latest”图像。Terrascan可以在图像中的“/go/bin”上找到，您可以根据需要使用任何Terrascan命令行选项。这里有一个例子。gitlab ci。yml文件：

```shell
stages:
  - scan

terrascan:
  image:
    name: accurics/terrascan:latest
    entrypoint: ["/bin/sh", "-c"]
  stage: scan
  script:
    - /go/bin/terrascan scan .

```





# 参考链接

* https://www.freebuf.com/articles/web/256898.html
* https://github.com/accurics/terrascan
* https://docs.accurics.com/projects/accurics-terrascan/en/latest/