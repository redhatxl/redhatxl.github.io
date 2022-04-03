# Docker 操作常用数据库

## 前言

对于后端开发人员，经常需要使用数据库，在本地安装数据库麻烦且易出错，利用docker能非常快速的拉启所需数据库环境，不用的时候可以删掉，如果需要本地存储数据可以使用单独数据目录挂在到容器内，本文简单列举几类常用数据库。



## 一 MySQL

```shell
# 拉取镜像
docker pull mysql:8.0.19

# 启动server
docker run --name mysql01 -p 13306:3306 -e MYSQL_ROOT_PASSWORD=mysqladmin -d mysql:8.0.19

# 启动客户端,输入密码：mysqladmin 
docker run -it --network host --rm mysql mysql -h127.0.0.1 -P13306 --default-character-set=utf8mb4 -uroot -p
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210929135446.png)

## 二 Redis

### 3.1 redis-server

```shell
# 拉取redis
docker pull redis
# 启动redis
docker run -itd --name redis01 -p 6379:6379 redis --requirepass "redisadmin" --appendonly yes

# 使用客户端链接redis
docker exec -it redis01 /bin/bash
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210929140512.png)

### 2.2 redis-cli

可以利用redis镜像直接使用其的cli进行操作。

```shell
docker run -itd --name redis-cli redis --requirepass "redisadmin" --appendonly yes
```



## 三 Etcd

```shell
# 拉去镜像
docker pull appcelerator/etcd:latest
# 启动
docker run --name etcd01 -d -p 2379:2379 -p 2380:2380 appcelerator/etcd --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://0.0.0.0:2379 

# 客户端链接
docker exec -it etcd01 /bin/bash
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210916143705.png)

## 四 Elasticsearch

```shell
# 拉取镜像
docker pull elasticsearch:latest
# 启动
docker run --name es01 -d -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" -e  "ES_JAVA_OPTS=-Xms1g -Xmx1g" elasticsearch:latest

# 使用head客户端链接
docker pull mobz/elasticsearch-head:5
# 启动header 容器
docker run -d --name my-es_admin -p 9100:9100 mobz/elasticsearch-head:5

# curl测试访问
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210916150444.png)

第一次打开浏览器header访问，连接的服务地址是localhost:9200，修改为docker所在的ip。此时出现连接失败，需要修改镜像的elasticsearch.yml文件，添加

```
http.cors.enabled: true
http.cors.allow-origin: "*"

# 重启es
docker restart es01
docker restart my-es-head
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210916151130.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211001203642.png)

## 五 MongoDB



```shell
#拉取镜像
docker pull mongo:lastest

# 启动
docker run --name mongodb01  -p 27017:27017 -d mongo:latest

# 客户端链接以admin进入容器
 docker exec -it mongodb01 mongo admin
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210916151926.png)

## 六 postgre

```shell
# 下载
docker pull postgres:12

# 启动
docker run --name pg01 -e POSTGRES_PASSWORD=pgadmin  -p 54320:5432 -d postgres:12


# 客户端链接
 docker exec -it pg01 /bin/bash
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210929143544.png)

## 其他 

本文通过利用Docker容器化封装的能力，将含有镜像直接从仓库拉取下来后，通过命令行运行，并将指定端口映射到本地。然后本地开发的时候，并不需要去关注数据库的配置和安装了，简单来说，就是查询镜像、拉取镜像、运行镜像。简单的三部操作就可以拥有一个配置好的需求数据库环境。



查看镜像版本：

```
curl https://registry.hub.docker.com/v1/repositories/{imagename}/tags
```

