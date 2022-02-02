# 4.Cobra+Client-go实现K8s插件开发

## 一 背景


在我们使用kubectl查看k8s资源的时候，想直接查看对应资源的容器名称和镜像名称，目前kubectl还不支持该选型，需要我们describe然后来查看，对于集群自己比较多，不是很方便，因此萌生自己开发kubectl 插件来实现该功能。


## 二 相关技术


### 2.1 Cobra


Cobra是一个命令行程序库，其是一个用来编写命令行的神器，提供了一个脚手架，用于快速生成基于Cobra应用程序框架。我们可以利用Cobra快速的去开发出我们想要的命令行工具，非常的方便快捷。


详细可参考：[Golang 开发之Cobra初探](https://juejin.cn/post/6983299467537547294)


### 2.2 Client-go


在K8s运维中，我们可以使用kubectl、客户端库或者REST请求来访问K8S API。而实际上，无论是kubectl还是客户端库，都是封装了REST请求的工具。client-go作为一个客户端库，能够调用K8S API，实现对K8S集群中资源对象（包括deployment、service、ingress、replicaSet、pod、namespace、node等）的增删改查等操作。


详细可参考：[K8s二开之 client-go 初探](https://juejin.cn/post/6962869412785487909)


### 2.3 k8s 插件krew


Krew 是 类似于系统的apt、dnf或者brew的 kubectl插件包管理工具，利用其可以轻松的完成kubectl 插件的全上面周期管理，包括搜索、下载、卸载等。


kubectl 其工具已经比较完善，但是对于一些个性化的命令，其宗旨是希望开发者能以独立而紧张形式发布自定义的kubectl子命令，插件的开发语言不限，需要将最终的脚步或二进制可执行程序以`kubectl-` 的前缀命名，然后放到PATH中即可，可以使用`kubectl plugin list`查看目前已经安装的插件。


详细可参考：[k8s 插件管理工具之krew使用](https://juejin.cn/post/6969183421381738533)


## 三 插件规划


- 插件命名为：kubeimg。
- 目前仅简单实现一个image命令，用于查看不同资源对象(deployments/daemonsets/statefulsets/jobs/cronjobs)的名称，和对应容器名称，镜像名称。
- 支持json格式输出。
- 最后将其作为krew插件使用。
- 可以直接根据名称空间来进行查看对应资源。

## 四 实战开发


### 4.1 项目初始化


- 安装cobra

```shell
go get -v github.com/spf13/cobra/cobra
```


- 初始化项目

```shell
 $ ~/workspace/goworkspace/src/github.com/kaliarch/kubeimg  /Users/xuel/workspace/goworkspace/bin/cobra init --pkg-name kubeimg
Your Cobra application is ready at
/Users/xuel/workspace/goworkspace/src/github.com/kaliarch/kubeimg
$ ~/workspace/goworkspace/src/github.com/kaliarch/kubeimg  ls
LICENSE cmd     main.go
$ ~/workspace/goworkspace/src/github.com/kaliarch/kubeimg  tree

├── LICENSE
├── cmd
│   └── root.go
└── main.go

1 directory, 3 files
```


- 创建go mod，下载相关包



```shell
go mod init kubeimg
```


### 4.2 增加子命令


增加一个子命令image。


```shell
$ /Users/xuel/workspace/goworkspace/bin/cobra add image
image created at /Users/xuel/workspace/goworkspace/src/github.com/kaliarch/kubeimg
```


### 4.3 添加参数


```go

// Execute adds all child commands to the root command and sets flags appropriately.
// This is called by main.main(). It only needs to happen once to the rootCmd.
func Execute() {
	cobra.CheckErr(rootCmd.Execute())
}

func init() {
	KubernetesConfigFlags = genericclioptions.NewConfigFlags(true)
	imageCmd.Flags().BoolP("deployments", "d", false, "show deployments image")
	imageCmd.Flags().BoolP("daemonsets", "e", false, "show daemonsets image")
	imageCmd.Flags().BoolP("statefulsets", "f", false, "show statefulsets image")
	imageCmd.Flags().BoolP("jobs", "o", false, "show jobs image")
	imageCmd.Flags().BoolP("cronjobs", "b", false, "show cronjobs image")
	imageCmd.Flags().BoolP("json", "j", false, "show json format")
	KubernetesConfigFlags.AddFlags(rootCmd.PersistentFlags())
}
```


### 4.4 实现image命令


```go
var imageCmd = &cobra.Command{
	Use:   "image",
	Short: "show resource image",
	Long:  `show k8s resource image`,
	RunE:  image,
}

func init() {
	rootCmd.AddCommand(imageCmd)
}
```


### 4.5 初始化clientset


```go
// ClientSet k8s clientset
func ClientSet(configFlags *genericclioptions.ConfigFlags) *kubernetes.Clientset {
	config, err := configFlags.ToRESTConfig()
	if err != nil {
		panic("kube config load error")
	}
	clientSet, err := kubernetes.NewForConfig(config)
	if err != nil {

		panic("gen kube config error")
	}
	return clientSet
}
```


### 4.6 实现查看资源对象


利用反射实现根据不同资源类型查看具体对应资源镜像及镜像名称功能。


```go
func image(cmd *cobra.Command, args []string) error {

	clientSet := kube.ClientSet(KubernetesConfigFlags)
	ns, _ := rootCmd.Flags().GetString("namespace")
	// 生命一个全局资源列表
	var rList []interface{}

	if flag, _ := cmd.Flags().GetBool("deployments"); flag {
		deployList, err := clientSet.AppsV1().Deployments(ns).List(context.Background(), v1.ListOptions{})
		if err != nil {
			fmt.Printf("list deployments error: %s", err.Error())
		}
		rList = append(rList, deployList)
	}
  ...
  	deployMapList := make([]map[string]string, 0)
	for i := 0; i < len(rList); i++ {
		switch t := rList[i].(type) {
		case *kv1.DeploymentList:
			for k := 0; k < len(t.Items); k++ {
				for j := 0; j < len(t.Items[k].Spec.Template.Spec.Containers); j++ {
					deployMap := make(map[string]string)
					deployMap["NAMESPACE"] = ns
					deployMap["TYPE"] = "deployment"
					deployMap["RESOURCE_NAME"] = t.Items[k].GetName()
					deployMap["CONTAINER_NAME"] = t.Items[k].Spec.Template.Spec.Containers[j].Name
					deployMap["IMAGE"] = t.Items[k].Spec.Template.Spec.Containers[j].Image
					deployMapList = append(deployMapList, deployMap)
				}
			}
```


### 4.6 实现输出


利用tabel来对结果进行输出


```go

func GenTable(mapList []map[string]string) *table.Table {
	t, err := gotable.Create(title...)
	if err != nil {
		fmt.Printf("create table error: %s", err.Error())
		return nil
	}
	t.AddRows(mapList)
	return t
}
```


最终项目结构：


![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210710233422.png)


## 五 测试


对完成的插件进行测试，编译`go build`生成kubeimg二进制可执行文件。


### 5.1 查看帮助


- 查看所有帮助

其中可以看到cobra帮我们自动生成了help和completion两个命令，可以快速实现table补全，支持bash/fish/zsh/powershell


```go
./kubeimg --help
```


![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210710224645.png)


![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210710225135.png)


- 查看image命令flags



```go
./kubeimg image --help
```


![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210710225248.png)


### 5.1 查看deployment资源


不知地ing名称名称空间，默认查看所有，名称空间下的资源


```shell
./kubeimg image -d
```


![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210710225937.png)


### 5.2 查看某个名称空间下资源


```shell
./kubeimg image -d -n kube-system
```


![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210710230804.png)


### 5.3 查看所有资源


可以看到imlc-operator-controller-manager 一个pod中有两个container。


```shell
./kubeimg image -b -e -d -o -f
```


![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210710230549.png)


### 5.4 json格式输出


```shell
./kubeimg image -o -j
```


![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210710231136.png)


## 六 作为krew插件使用


需要将最终的脚步或二进制可执行程序以`kubectl-` 的前缀命名，然后放到PATH中即可，可以使用`kubectl plugin list`查看目前已经安装的插件。


```shell
$ kubectl plugin list
The following compatible plugins are available:=
/usr/local/bin/kubectl-debug
  - warning: kubectl-debug overwrites existing command: "kubectl debug"
/usr/local/bin/kubectl-v1.10.11
/usr/local/bin/kubectl-v1.20.0
/Users/xuel/.krew/bin/kubectl-df_pv
/Users/xuel/.krew/bin/kubectl-krew

# 将自己开发的插件重新命名为kubectl-img放到可执行路基下
$ cp kubeimg /Users/xuel/.krew/bin/kubectl-img

$ kubectl plugin list
The following compatible plugins are available:=
/usr/local/bin/kubectl-debug
  - warning: kubectl-debug overwrites existing command: "kubectl debug"
/usr/local/bin/kubectl-v1.10.11
/usr/local/bin/kubectl-v1.20.0
/Users/xuel/.krew/bin/kubectl-df_pv
/Users/xuel/.krew/bin/kubectl-krew
/Users/xuel/.krew/bin/kubectl-img

$ cp kubeimg /Users/xuel/.krew/bin/kubectl-img
```


之后就可以想使用kubectl插件一样使用了。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/Jietu20210710-232431.gif)




## 其他


目前实现的比较简单，以此来抛砖引玉的功能，后期可以进行更多功能或其他插件的开发，自己动手丰衣足食。


目前已开源到github，项目地址：[kubectl-img](https://github.com/redhatxl/kubectl-img)，以供大家学习交流。


## 参考链接


- [https://juejin.cn/post/6969183421381738533](https://juejin.cn/post/6969183421381738533)
- [https://juejin.cn/post/6962869412785487909](https://juejin.cn/post/6962869412785487909)
- [https://github.com/spf13/cobra](https://github.com/spf13/cobra)
