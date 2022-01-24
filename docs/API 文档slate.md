# API 文档slate

## 1 安装部署

### 1.1 ruby安装

* 安装RVM

[rvm官网]([https://rvm.io/](https://link.jianshu.com/?t=https%3A%2F%2Frvm.io%2F))

```shell
curl -sSL https://get.rvm.io | bash -s stable
```

![image-20190702092519861](/Users/xuel/Library/Application Support/typora-user-images/image-20190702092519861.png)

* 检查

```shell
 xuel@xueleideMacBook-Pro  ~  rvm -v
zsh: command not found: rvm
 ✘ xuel@xueleideMacBook-Pro  ~  source .rvm/scripts/rvm
 xuel@xueleideMacBook-Pro  ~  rvm -v
rvm 1.29.8 (latest) by Michal Papis, Piotr Kuczynski, Wayne E. Seguin [https://rvm.io]
#修改 RVM 下载 Ruby 的源，到 Ruby China 的镜像:
 xuel@xueleideMacBook-Pro  ~  echo "ruby_url=https://cache.ruby-china.org/pub/ruby" > ~/.rvm/user/db
 xuel@xueleideMacBook-Pro  ~  cat .rvm/user/db
ruby_url=https://cache.ruby-china.org/pub/ruby
 xuel@xueleideMacBook-Pro  ~  rvm -v
rvm 1.29.8 (latest) by Michal Papis, Piotr Kuczynski, Wayne E. Seguin [https://rvm.io]
```

* 更新

```shell
rvm get stable
```

### 1.2 ruby安装

* 查看ruby列表

```shel
 
 xuel@xueleideMacBook-Pro  ~  rvm list known
 # 安装
 xuel@xueleideMacBook-Pro  ~  rvm install 2.6 --default
 
 # 查看版本
  xuel@xueleideMacBook-Pro  ~  ruby -v
ruby 2.6.3p62 (2019-04-16 revision 67580) [x86_64-darwin18]
```











## 相关链接

https://github.com/lord/slate