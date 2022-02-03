# Gin框架热加载

# 一 背景

Air能够实时监听项目的代码文件，在代码发生变更之后自动重新编译并执行，大大提高gin框架项目的开发效率。

## 1.2 什么是热加载

如果你是一名python开发者，应该很熟悉这个。我们在Flask或者Django框架下开发都是支持实时加载的，当我们对代码进行修改时，程序能够自动重新加载并执行，这在我们开发中是非常便利的，可以快速进行代码测试，省去了每次手动重新编译。

如果你是一名JAVA开发者，不仅会听过热加载，热部署会跟着一块出现。热部署一般是指容器（支持多应用）不重启，单独启动单个应用。热加载一般指重启应用（JVM），单独重新更新某个类或者配置文件。

# 二 实现方案

## 2.1 使用fresh实现

```shell
	go get github.com/pilu/fresh
	# 跳转到要运行的项目的 根目录 ，运行 fresh 命令 。fresh 将会自动运行项目的 main.go  
	fresh
	
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210921224547.png)

## 2.2 使用air实现

参考：https://www.liwenzhou.com/posts/Go/live_reload_with_air/

# 三 实战





# 四 反思



# 参考链接

* https://www.liwenzhou.com/posts/Go/live_reload_with_air/	
* https://studygolang.com/articles/30231?fr=sidebar