# Kubernetes安全扫描之kubescape

## 一 背景

Kubescape 是第一个用于测试 Kubernetes 是否按照 NSA 和 CISA 的 Kubernetes 强化指南中定义的安全部署的工具 使用 Kubescape 测试集群或扫描单个 YAML 文件并将其集成到您的流程中

## 二 安装

```shell
#!/bin/bash
set -e

echo "Installing Kubescape..."
echo
 
BASE_DIR=~/.kubescape
KUBESCAPE_EXEC=kubescape

osName=$(uname -s)
if [[ $osName == *"MINGW"* ]]; then
    osName=windows-latest
elif [[ $osName == *"Darwin"* ]]; then
    osName=macos-latest
else
    osName=ubuntu-latest
fi

GITHUB_OWNER=armosec

DOWNLOAD_URL=$(curl --silent "https://api.github.com/repos/$GITHUB_OWNER/kubescape/releases/latest" | grep -o "browser_download_url.*${osName}.*")
DOWNLOAD_URL=${DOWNLOAD_URL//\"}
DOWNLOAD_URL=${DOWNLOAD_URL/browser_download_url: /}

mkdir -p $BASE_DIR 

OUTPUT=$BASE_DIR/$KUBESCAPE_EXEC

curl --progress-bar -L $DOWNLOAD_URL -o $OUTPUT
echo -e "\033[32m[V] Downloaded Kubescape"

# Ping download counter
curl --silent https://us-central1-elated-pottery-310110.cloudfunctions.net/kubescape-download-counter -o /dev/null
 
chmod +x $OUTPUT || sudo chmod +x $OUTPUT
rm -f /usr/local/bin/$KUBESCAPE_EXEC || sudo rm -f /usr/local/bin/$KUBESCAPE_EXEC
cp $OUTPUT /usr/local/bin || sudo cp $OUTPUT /usr/local/bin
rm -rf $OUTPUT

echo -e "[V] Finished Installation"
echo

echo -e "\033[35m Usage: $ $KUBESCAPE_EXEC scan framework nsa --exclude-namespaces kube-system,kube-public"
echo
```

## 三 监测

```shell
chmod +x /root/.kubescape/kubescape 
/root/.kubescape/kubescape scan framework nsa --exclude-namespaces kube-system,kube-public
```

* 输出结果

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210918094520.png)

## 参考链接

* https://github.com/armosec/kubescape