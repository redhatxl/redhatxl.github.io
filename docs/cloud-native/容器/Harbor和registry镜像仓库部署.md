# Harbor/registry镜像仓库部署

## 一 Harbor部署

### 1.1 安装docker

```shell
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum install -y docker-ce yum-utils device-mapper-persistent-data lvm2
systemctl start docker

cat > /etc/docker/daemon.json << EOF
{
"exec-opts": ["native.cgroupdriver=systemd"],
"log-driver": "json-file",
"log-opts": {
"max-size": "100m"
}
}
EOF

systemctl enable docker
```

### 1.2 安装docker-compose

```shell
systemctl enable docker

// 安装docker-compose
wget https://github.com/docker/compose/releases/download/1.29.2/docker-compose-Linux-x86_64
chmod +x docker-compose-Linux-x86_64
mv docker-compose-Linux-x86_64 docker-compose
mv docker-compose /usr/bin
```

### 1.3 下载harbor

进入harbor目录，修改harbor.yml配置文件，软件自带有一个tmpl的模板文件，可以通过复制此文件进行编辑。

```shell
wget https://github.com/goharbor/harbor/releases/download/v2.2.2/harbor-offline-installer-v2.2.2.tgz
tar -zxf harbor-offline-installer-v2.2.2.tgz
cd harbor
cp harbor.yml.tmpl harbor.yml

vim harbor.yml

# 修改完成后进行安装
./install.sh
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210611172822.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210611172754.png)

### 1.4 停止和启动

因为Harbor是基于docker-compose服务编排的，所以通过 docker-compose启动或者关闭Harbor

```shell
docker-compose down

docker-compose up -d
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210611173214.png)

## 二 registry

docker hub 公共镜像中有 registry 的镜像，直接从docker hub拉取。此操作在192.168.1.201上执行

```shell
mkdir /registry
docker run -p 5000:5000  --restart=always --name registry -v /registry/:/var/lib/registry -d registry

[root@img-tools registry]# docker pull centos:latest
[root@img-tools registry]# docker tag centos:latest 127.0.0.1:5000/mycentos:latest
[root@img-tools registry]# docker push 127.0.0.1:5000/mycentos:latest
The push refers to repository [127.0.0.1:5000/mycentos]
2653d992f4ef: Pushed
latest: digest: sha256:dbbacecc49b088458781c16f3775f2a2ec7521079034a7ba499c8b0bb7f86875 size: 529
[root@img-tools registry]# curl -X GET http://127.0.0.1:5000/v2/_catalog -k
{"repositories":["mycentos"]}

// 查看获取到镜像的Digest
[root@img-tools harbor]# curl -I -H "Accept: application/vnd.docker.distribution.manifest.v2+json" localhost:5000/v2/mycentos/manifests/latest
HTTP/1.1 200 OK
Content-Length: 529
Content-Type: application/vnd.docker.distribution.manifest.v2+json
Docker-Content-Digest: sha256:dbbacecc49b088458781c16f3775f2a2ec7521079034a7ba499c8b0bb7f86875
Docker-Distribution-Api-Version: registry/2.0
Etag: "sha256:dbbacecc49b088458781c16f3775f2a2ec7521079034a7ba499c8b0bb7f86875"
X-Content-Type-Options: nosniff
Date: Fri, 11 Jun 2021 09:15:55 GMT

// 查看宿主机信息

[root@img-tools ~]# tree /registry/docker/
/registry/docker/
└── registry
    └── v2
        ├── blobs
        │   └── sha256
        │       ├── 30
        │       │   └── 300e315adb2f96afe5f0b2780b87f28ae95231fe3bdd1e16b9ba606307728f55
        │       │       └── data
        │       ├── 7a
        │       │   └── 7a0437f04f83f084b7ed68ad9c4a4947e12fc4e1b006b38129bac89114ec3621
        │       │       └── data
        │       └── db
        │           └── dbbacecc49b088458781c16f3775f2a2ec7521079034a7ba499c8b0bb7f86875
        │               └── data
        └── repositories
            └── mycentos
                ├── _layers
                │   └── sha256
                │       ├── 300e315adb2f96afe5f0b2780b87f28ae95231fe3bdd1e16b9ba606307728f55
                │       │   └── link
                │       └── 7a0437f04f83f084b7ed68ad9c4a4947e12fc4e1b006b38129bac89114ec3621
                │           └── link
                ├── _manifests
                │   ├── revisions
                │   │   └── sha256
                │   │       └── dbbacecc49b088458781c16f3775f2a2ec7521079034a7ba499c8b0bb7f86875
                │   │           └── link
                │   └── tags
                │       └── latest
                │           ├── current
                │           │   └── link
                │           └── index
                │               └── sha256
                │                   └── dbbacecc49b088458781c16f3775f2a2ec7521079034a7ba499c8b0bb7f86875
                │                       └── link
                └── _uploads

27 directories, 8 files


[root@img-tools registry]# curl -X GET http://127.0.0.1:5000/v2/_catalog -k
{"repositories":["mycentos"]}
[root@img-tools registry]# curl -X GET http://127.0.0.1:5000/v2/mycentos/tags/list
{"name":"mycentos","tags":["latest"]}



# 启用证书
mkdir -p /opt/docker/registry/certs
openssl req -newkey rsa:4096 -nodes -sha256 -keyout /opt/docker/registry/certs/domain.key -x509 -days 365 -out /opt/docker/registry/certs/domain.crt

[root@img-tools imgtools]# ll /opt/docker/registry/certs/
总用量 8
-rw-r--r-- 1 root root 2090 6月  11 18:10 domain.crt
-rw-r--r-- 1 root root 3272 6月  11 18:10 domain.key


docker run -p 5000:5000 --restart=always --name myregistry -v /registry/:/var/lib/registry -v /opt/docker/registry/certs:/certs -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key -d registry


# 进行测试
[root@img-tools registry]# curl -X GET https://registryxl.com:5000/v2/mycentos/tags/list -k
{"name":"mycentos","tags":["latest"]}


# 添加基本验证
mkdir /opt/docker/registry/auth/
[root@img-tools registry]# htpasswd -Bbn admin admin > /opt/docker/registry/auth/htpasswd
[root@img-tools registry]# cat /opt/docker/registry/auth/htpasswd
admin:$2y$05$7Jy0VmD3I3HDBczRBgcoa.A8Kz4rxdtlM5lx5GrHdCSDk.DrSnUXO


启动带认证的 Docker Registry
REGISTRY_AUTH=htpasswd # 以 htpasswd 的方式认证
REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm # 注册认证
REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd # 认证的用户密码

# 启动 带不可信证书，待用户认证
docker run -p 5000:5000 --restart=always --name myregistry -v /registry/:/var/lib/registry -v /opt/docker/registry/certs:/certs -v /opt/docker/registry/auth/:/auth/ -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key -e "REGISTRY_AUTH=htpasswd" -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" -e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd"  -d registry


# 不带https，仅待用户认证
docker run -p 5000:5000 --restart=always --name myregistry -v /registry/:/var/lib/registry -v /opt/docker/registry/auth/:/auth/  -e "REGISTRY_AUTH=htpasswd" -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" -e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd"  -d registry

# 不安全，
[root@img-tools ~]# curl -X GET registryxl.com:5000/v2/_catalog -uadmin
Enter host password for user 'admin':
{"repositories":["mycentos"]}


[root@img-tools registry]# docker login https://registryxl.com:5000
Username: admin
Password:
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded



[root@img-tools ~]# curl -X GET https://registryxl.com:5000/v2/_catalog -u admin -k
Enter host password for user 'admin':
{"repositories":["mycentos"]}
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210611181543.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210611181835.png)

# 参考链接

* https://blog.csdn.net/wennuanddianbo/article/details/95205644

* https://www.cnblogs.com/chen2ha/p/14787695.html