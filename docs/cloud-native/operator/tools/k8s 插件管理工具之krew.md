# k8s 插件管理工具之krew

  ## 一 简介

  Krew 是 类似于系统的apt、dnf或者brew的 kubectl插件包管理工具，利用其可以轻松的完成kubectl 插件的全上面周期管理，包括搜索、下载、卸载等。

  kubectl 其工具已经比较完善，但是对于一些个性化的命令，其宗旨是希望开发者能以独立而紧张形式发布自定义的kubectl子命令，插件的开发语言不限，需要将最终的脚步或二进制可执行程序以`kubectl-` 的前缀命名，然后放到PATH中即可，可以使用`kubectl plugin list`查看目前已经安装的插件。

  ## 二 安装配置

  * 确保节点安装有git工具

  ```shell
  # yum -y install git
  ```

  * 安装

  ```shell
  (
    set -x; cd "$(mktemp -d)" &&
    OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
    ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
    curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/krew.tar.gz" &&
    tar zxvf krew.tar.gz &&
    KREW=./krew-"${OS}_${ARCH}" &&
    "$KREW" install krew
  )
  ```

  * 添加环境变量

  ```shell
  export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
  source ~/.bashrc
  ```

  * 确认插件安装

  ```shell
   # kubectl plugin list
  /usr/local/bin/kubectl-debug
  /usr/local/bin/kubectl-v1.10.11
  /usr/local/bin/kubectl-v1.20.0
  /Users/xuel/.krew/bin/kubectl-df_pv
  /Users/xuel/.krew/bin/kubectl-krew
  // 或者


  ```

  ## 三 使用

  ```shell
  kubectl krew update								# 更新
  kubectl krew search               # 显示所有插件
  kubectl krew install view-secret  # 安装名为view-secret的插件
  kubectl view-secret               # 使用该插件
  kubectl krew upgrade              # 升级安装的插件
  kubectl krew remove view-secret   # 卸载插件
  ```

  krew自身也作为一个“kubectl 插件”，因此，可以使用命令`kubectl krew upgrade`命令来升级krew。

  ### 3.1 安装插件

  kubectl 无法直接查看pv的大小相关信息，可以安装一个查看pv大小的插件

  ```shell
  # kubectl krew install df-pv
  Updated the local copy of plugin index.
  Installing plugin: df-pv
  Installed plugin: df-pv
  \
   | Use this plugin:
   | 	kubectl df-pv
   | Documentation:
   | 	https://github.com/yashbhutwala/kubectl-df-pv

  #  kubectl krew list
  PLUGIN  VERSION
  df-pv   v0.2.7
  krew    v0.4.1
  ```

  ### 3.2 使用

  ```shell
  # kubectl df-pv
  ```

  ![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210602132921.png)

  ### 3.3 卸载

  ```shell
  kubectl krew uninstall df-pv
  ```

  ## 四 其他

   K8s社区为方便其他开发这开发插件，提供了一个https://github.com/kubernetes/cli-runtime项目，便于我们使用Go语言编写kubectl插件。 

  官方也给了一个使用Go编写kubectl插件的例子https://github.com/kubernetes/sample-cli-plugin。

  krew 仅仅兼容kubectl v1.12或更高版本。

  ## 参考链接

  * https://krew.sigs.k8s.io/docs/user-guide/quickstart/

  * https://github.com/kubernetes-sigs/krew/blob/master/docs/KREW_ARCHITECTURE.md
  * https://github.com/kubernetes-sigs/krew
  * https://blog.51cto.com/u_3241766/2452592



