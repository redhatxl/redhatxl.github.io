## 1.1 概述

近期进行业务容器后改造，部署K8S需要为其提供存储，，在选型中本地存储不可跨node，NFS共享存储不好做高可用，因此选型Ceph来为k8s提供存储类，以备后用
Ceph是一种为优秀的性能、可靠性和可扩展性而设计的统一的、分布式文件系统。Ceph是一个开源的分布式文件系统。因为它还支持块存储、对象存储，所以很自然的被用做云计算框架openstack或cloudstack整个存储后端。当然也可以单独作为存储，例如部署一套集群作为对象存储、SAN存储、NAS存储等。可以作为k8s的存储类，来方便容器持久化存储。

## 1.2 支持格式

* 对象存储：即radosgw,兼容S3接口。通过rest api上传、下载文件。

* 文件系统：posix接口。可以将ceph集群看做一个共享文件系统挂载到本地。

* 块存储：即rbd。有kernel rbd和librbd两种使用方式。支持快照、克隆。相当于一块硬盘挂到本地，用法和用途和硬盘一样。比如在OpenStack项目里，Ceph的块设备存储可以对接OpenStack的后端存储

## 1.3 优势

* 统一存储：虽然ceph底层是一个分布式文件系统，但由于在上层开发了支持对象和块的接口

* 高扩展性：扩容方便、容量大。能够管理上千台服务器、EB级的容量。

* 高可靠性：支持多份强一致性副本，EC。副本能够垮主机、机架、机房、数据中心存放。所以安全可靠。存储节点可以自管理、自动修复。无单点故障，容错性强。

* 高性能：因为是多个副本，因此在读写操作时候能够做到高度并行化。理论上，节点越多，整个集群的IOPS和吞吐量越高。另外一点ceph客户端读写数据直接与存储设备(osd) 交互。

## 1.4 核心组件

* Ceph OSDs:Ceph OSD 守护进程（ Ceph OSD ）的功能是存储数据，处理数据的复制、恢复、回填、再均衡，并通过检查其他OSD 守护进程的心跳来向 Ceph Monitors 提供一些监控信息。当 Ceph 存储集群设定为有2个副本时，至少需要2个 OSD 守护进程，集群才能达到 active+clean 状态（ Ceph 默认有3个副本，但你可以调整副本数）。

* Monitors:  Ceph Monitor维护着展示集群状态的各种图表，包括监视器图、 OSD 图、归置组（ PG ）图、和 CRUSH 图。 Ceph 保存着发生在Monitors 、 OSD 和 PG上的每一次状态变更的历史信息（称为 epoch ）。

* MDSs:  Ceph 元数据服务器（ MDS ）为 Ceph 文件系统存储元数据（也就是说，Ceph 块设备和 Ceph 对象存储不使用MDS ）。元数据服务器使得 POSIX 文件系统的用户们，可以在不对 Ceph 存储集群造成负担的前提下，执行诸如 ls、find 等基本命令。



# 二 安装部署

## 2.1 主机信息

| 主机名 | 操作系统         | 配置            | K8S组件 | CEPH组件       | 私网IP      | SSH端口 | 用户名密码            |
| ------ | ---------------- | --------------- | ------- | -------------- | ----------- | ------- | --------------------- |
| master | CentOS 7.4 64bit | 4C8G + 500G硬盘 |         | admin,osd, mon | 172.16.60.2 | 2001/22 | root/uWWKWnjySO7Zocuh |
| node01 | CentOS 7.4 64bit | 4C8G + 500G硬盘 |         | osd, mon       | 172.16.60.3 | 2002/22 | root/IZ5lReaUBz3QOkLh |
| node02 | CentOS 7.4 64bit | 4C8G + 500G硬盘 |         | osd, mon       | 172.16.60.4 | 2003/22 | root/nUMFlg9a4zpzDMcE |

## 2.2 磁盘准备

需要在三台主机创建磁盘,并挂载到主机的/var/local/osd{0,1,2}

```shell
[root@master ~]# mkfs.xfs /dev/vdc
[root@master ~]# mkdir -p /var/local/osd0
[root@master ~]# mount /dev/vdc /var/local/osd0/


[root@node01 ~]# mkfs.xfs /dev/vdc
[root@node01 ~]# mkdir -p /var/local/osd1
[root@node01 ~]# mount /dev/vdc /var/local/osd1/

[root@node02 ~]# mkfs.xfs /dev/vdc 
[root@node02 ~]# mkdir -p /var/local/osd2
[root@node02 ~]# mount /dev/vdc /var/local/osd2/

将磁盘添加进入fstab中，确保开机自动挂载

```

## 2.3 配置各主机hosts文件

```shell
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.16.60.2 master
172.16.60.3 node01
172.16.60.4 node02
```

## 2.4 管理节点ssh免密钥登录node1/node2

```shell
[root@master ~]# ssh-keygen -t rsa
[root@master ~]# ssh-copy-id -i /root/.ssh/id_rsa.pub root@node01
[root@master ~]# ssh-copy-id -i /root/.ssh/id_rsa.pub root@node02
```

## 2.5 master节点安装ceph-deploy工具

```shell
# 各节点均更新ceph的yum源
vim /etc/yum.repos.d/ceph.repo 

[ceph]
name=ceph
baseurl=http://mirrors.aliyun.com/ceph/rpm-jewel/el7/x86_64/
gpgcheck=0
priority =1
[ceph-noarch]
name=cephnoarch
baseurl=http://mirrors.aliyun.com/ceph/rpm-jewel/el7/noarch/
gpgcheck=0
priority =1
[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-jewel/el7/SRPMS
gpgcheck=0
priority=1

# 安装ceph-deploy工具
yum clean all && yum makecache
yum -y install ceph-deploy
```

## 2.6 创建monitor服务

创建monitor服务,指定master节点的hostname

```shell
[root@master ~]# mkdir /etc/ceph && cd /etc/ceph
[root@master ceph]# ceph-deploy new master
[root@master ceph]# ll
total 12
-rw-r--r-- 1 root root  195 Sep  3 10:56 ceph.conf
-rw-r--r-- 1 root root 2915 Sep  3 10:56 ceph-deploy-ceph.log
-rw------- 1 root root   73 Sep  3 10:56 ceph.mon.keyring


[root@master ceph]# cat ceph.conf 
[global]
fsid = 5b9eb8d2-1c12-4f6d-ae9c-85078795794b
mon_initial_members = master
mon_host = 172.16.60.2
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
osd_pool_default_size = 2

配置文件的默认副本数从3改成2，这样只有两个osd也能达到active+clean状态，把下面这行加入到[global]段（可选配置）
```

## 2.7 所有节点安装ceph

```shell
# 各节点安装软件包
yum -y install yum-plugin-priorities epel-release
# master节点利用ceph-deply 部署ceph

[root@master ceph]# ceph-deploy install master node01 node02

[root@master ceph]# ceph --version
ceph version 10.2.11 (e4b061b47f07f583c92a050d9e84b1813a35671e)
```

## 2.8 部署相关服务

```shell
# 安装ceph monitor
[root@master ceph]# ceph-deploy mon create master

# 收集节点的keyring文件
[root@master ceph]# ceph-deploy  gatherkeys master

# 创建osd
[root@master ceph]# ceph-deploy osd prepare master:/var/local/osd0 node01:/var/local/osd1 node02:/var/local/osd2

# 权限修改
[root@master ceph]# chmod 777 -R /var/local/osd{0..2}
[root@master ceph]# chmod 777 -R /var/local/osd{0..2}/*

# 激活osd
[root@master ceph]# ceph-deploy osd activate master:/var/local/osd0 node01:/var/local/osd1 node02:/var/local/osd2

# 查看状态
[root@master ceph]# ceph-deploy osd list master node01 node02
```

## 2.9 统一配置

用ceph-deploy把配置文件和admin密钥拷贝到所有节点，这样每次执行Ceph命令行时就无需指定monitor地址和ceph.client.admin.keyring了



```shell
[root@master ceph]# ceph-deploy admin master node01 node02

# 各节点修改ceph.client.admin.keyring权限：
[root@master ceph]# chmod +r /etc/ceph/ceph.client.admin.keyring


# 查看状态
[root@master ceph]# ceph health
HEALTH_OK
[root@master ceph]# ceph -s
    cluster 5b9eb8d2-1c12-4f6d-ae9c-85078795794b
     health HEALTH_OK
     monmap e1: 1 mons at {master=172.16.60.2:6789/0}
            election epoch 3, quorum 0 master
     osdmap e15: 3 osds: 3 up, 3 in
            flags sortbitwise,require_jewel_osds
      pgmap v27: 64 pgs, 1 pools, 0 bytes data, 0 objects
            15681 MB used, 1483 GB / 1499 GB avail
                  64 active+clean
                      
```

## 2.10 部署MDS服务

我们在node01/node02上安装部署MDS服务

```shell
[root@master ceph]# ceph-deploy mds create node01 node02

# 查看状态
[root@master ceph]# ceph mds stat
e3:, 2 up:standby
[root@master ~]# ceph mon stat
e1: 1 mons at {master=172.16.60.2:6789/0}, election epoch 4, quorum 0 master

# 查看服务
[root@master ceph]# systemctl list-unit-files |grep ceph
ceph-create-keys@.service                     static  
ceph-disk@.service                            static  
ceph-mds@.service                             disabled
ceph-mon@.service                             enabled 
ceph-osd@.service                             enabled 
ceph-radosgw@.service                         disabled
ceph-mds.target                               enabled 
ceph-mon.target                               enabled 
ceph-osd.target                               enabled 
ceph-radosgw.target                           enabled 
ceph.target                                   enabled 
```

至此，基本上完成了ceph存储集群的搭建。 

# 三 创建ceph文件系统

## 3.1 创建文件系统

关于创建存储池
确定 pg_num 取值是强制性的，因为不能自动计算。下面是几个常用的值：
* 少于 5 个 OSD 时可把 pg_num 设置为 128
* OSD 数量在 5 到 10 个时，可把 pg_num 设置为 512
* OSD 数量在 10 到 50 个时，可把 pg_num 设置为 4096
* OSD 数量大于 50 时，你得理解权衡方法、以及如何自己计算 pg_num 取值
* 自己计算 pg_num 取值时可借助 pgcalc 工具
　　随着 OSD 数量的增加，正确的 pg_num 取值变得更加重要，因为它显著地影响着集群的行为、以及出错时的数据持久性（即灾难性事件导致数据丢失的概率）。 

```shell
[root@master ceph]# ceph osd pool create cephfs_data <pg_num> 
[root@master ceph]# ceph osd pool create cephfs_metadata <pg_num>

[root@master ~]# ceph osd pool ls 
rbd
[root@master ~]#  ceph osd pool create kube 128
pool 'kube' created
[root@master ~]# ceph osd pool ls              
rbd
kube

# 查看证书
[root@master ~]# ceph auth list
installed auth entries:

mds.node01
        key: AQB56m1dE42rOBAA0yRhsmQb3QMEaTsQ71jHdg==
        caps: [mds] allow
        caps: [mon] allow profile mds
        caps: [osd] allow rwx
mds.node02
        key: AQB66m1dWuhWKhAAtbiZN7amGcjUh6Rj/HNFkg==
        caps: [mds] allow
        caps: [mon] allow profile mds
        caps: [osd] allow rwx
osd.0
        key: AQA46W1daFx3IxAAE1esQW+t1fWJDfEQd+167w==
        caps: [mon] allow profile osd
        caps: [osd] allow *
osd.1
        key: AQBA6W1daJG9IxAAQwETgrVc3awkEZejDSaaow==
        caps: [mon] allow profile osd
        caps: [osd] allow *
osd.2
        key: AQBI6W1dot4/GxAAle3Ii3/D38RmwNC4yTCoPg==
        caps: [mon] allow profile osd
        caps: [osd] allow *
client.admin
        key: AQBu4W1d90dZKxAAH/kta03cP5znnCcWeOngzQ==
        caps: [mds] allow *
        caps: [mon] allow *
        caps: [osd] allow *
client.bootstrap-mds
        key: AQBv4W1djJ1uHhAACzBcXjVoZFgLg3lN+KEv8Q==
        caps: [mon] allow profile bootstrap-mds
client.bootstrap-mgr
        key: AQCS4W1dna9COBAAiWPu7uk3ItJxisVIwn2duA==
        caps: [mon] allow profile bootstrap-mgr
client.bootstrap-osd
        key: AQBu4W1dxappOhAA5FanGhQhAOUlizqa5uMG3A==
        caps: [mon] allow profile bootstrap-osd
client.bootstrap-rgw
        key: AQBv4W1dpwvsDhAAyp58v08XttJWzLoHWVHZow==
        caps: [mon] allow profile bootstrap-rgw
```



## 3.2 创建客户端密钥

```shell
# 创建keyring
[root@master ~]# ceph auth get-or-create client.kube mon 'allow r' osd 'allow rwx pool=kube' -o /etc/ceph/ceph.client.kube.keyring
[root@master ~]# ceph auth list

# 将密钥拷贝到node1和node2
[root@master ceph]# scp ceph.client.kube.keyring root@node01:/etc/ceph/
```

## 3.3 部署进k8s

```yaml
[root@master storageclass]# cat ceph-secret.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: ceph-admin-secret
  namespace: kube-system
type: "kubernetes.io/rbd"
data:
  # ceph auth get-key client.admin |base64
  key: QVFCRitmUmM1c1FuR3hBQUtBeFl5cFNJaW4wbHFoQVh6NlRvQ2c9PQ==

---
apiVersion: v1
kind: Secret
metadata:
  name: ceph-client-secret
  namespace: kube-system
type: "kubernetes.io/rdb"
data:
  # ceph auth get-key client.kube |base64
  key: QVFCMEovVmM2a3laSGhBQUxkOEszQlJEUlQyaGk0b09FSnJZL1E9PQ==	
  
  
[root@master storageclass]# cat ceph-storageclass.yaml 
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rdb
provisioner: ceph.com/rbd
reclaimPolicy: Retain
parameters:
  monitors: 10.234.2.204:6789
  pool: kube
  adminId: admin
  adminSecretName: ceph-admin-secret
  adminSecretNamespace: kube-system
  userId: kube
  userSecretName: ceph-client-secret
  userSecretNamespace: kube-system
  fsType: xfs
  imageFormat: "2"
  imageFeatures: "layering"
  
[root@master storageclass]# kubectl get sc
NAME       PROVISIONER    AGE
ceph-rdb   ceph.com/rbd   2m13s
```

​	

·

# 四 卸载

```shell
清理机器上的ceph相关配置：
停止所有进程： stop ceph-all
卸载所有ceph程序：ceph-deploy uninstall [{ceph-node}]
删除ceph相关的安装包：ceph-deploy purge {ceph-node} [{ceph-data}]
删除ceph相关的配置：ceph-deploy purgedata {ceph-node} [{ceph-data}]
删除key：ceph-deploy forgetkeys

卸载ceph-deploy管理：yum -y remove ceph-deploy
```



# 五 Cephfs

## 5.1 背景

因为上一篇文档中描述的使用RBD模式创建的pvc不支持RWM(readwriteMany)，只支持 RWO(ReadWriteOnce)和ROM(ReadOnlyMany)

k8s 不支持跨节点挂载同一Ceph RBD，支持跨节点挂载 CephFS，让所有的任务都调度到指定node上执行，来保证针对RBD的操作在同一节点上。

## 5.2 创建cephfs

在ceph服务器创建cephfs文件系统

```shell
ceph osd pool create fs_kube_data 128
ceph osd pool create fs_kube_metadata 128
ceph fs new cephfs fs_kube_metadata fs_kube_data
```

## 5.3 cephf 作为k8s的storage class

官方支持storage class的存储类型里，目前没有cephfs。目前k8s（v1.10）的确是不支持cephfs用作storage class的，但是我们已经有了ceph集群，而RBD又不支持 ReadWriteMany，这样就没办法做共享存储类型的Volume了，使用场景会受一些限制。而为了这个场景再单独去维护一套glusterfs，又很不划算，当然是能用cephfs做storage class最完美啦。
  社区其实是有项目做这个的，就是 external storage，只是现在还在孵化器里。所谓external storage其实就是一个 controller，它会去监听 apiserver 的storage class api的变化，当发现有cephfs的请求，它会拿来处理：根据用户的请求，创建PV，并将PV的创建请求发给api server；PV创建后再将其与PVC绑定。
  同时，controller的 cephfs-provisioner会调用 cephfs_provisioner.py 脚本去cephfs集群上去创建目录，目录名规则为 "kubernetes-dynamic-pvc-%s", uuid.NewUUID()。 也就是说，external-storage作为ceph集群的一个用户，自己维护了不同PVC和cephfs目录的对应关系。

```yaml
# 创建文件系统
[root@master cephfsdemo]#ceph osd pool create fs_kube_data 128
[root@master cephfsdemo]#ceph osd pool create fs_kube_metadata 128
[root@master cephfsdemo]#ceph fs new cephfs fs_kube_metadata fs_kube_data

[root@master cephfsdemo]# ceph fs ls
name: cephfs, metadata pool: fs_kube_metadata, data pools: [fs_kube_data ]
# 部署进k8s
[root@master deploy]# git clone https://github.com/kubernetes-incubator/external-storage.git
[root@master deploy]# cd /dockerdata/workspace/storageclass/external-storage-master/ceph/cephfs/deploy
[root@master deploy]# ls
non-rbac  rbac  README.md

# README.md内有启用rbac的安装方式
[root@master deploy]# kubectl create ns cephfs
[root@master deploy]# NAMESPACE=cephfs
[root@master deploy]# sed -r -i "s/namespace: [^ ]+/namespace: $NAMESPACE/g" ./rbac/*.yaml
[root@master deploy]# sed -r -i "N;s/(name: PROVISIONER_SECRET_NAMESPACE.*\n[[:space:]]*)value:.*/\1value: $NAMESPACE/" ./rbac/deployment.yaml
[root@master deploy]# kubectl -n $NAMESPACE apply -f ./rbac
[root@master deploy]# kubectl -n $NAMESPACE apply -f ./rbac
clusterrole.rbac.authorization.k8s.io/cephfs-provisioner unchanged
clusterrolebinding.rbac.authorization.k8s.io/cephfs-provisioner unchanged
deployment.extensions/cephfs-provisioner created
role.rbac.authorization.k8s.io/cephfs-provisioner created
rolebinding.rbac.authorization.k8s.io/cephfs-provisioner created
serviceaccount/cephfs-provisioner created

# 待pod起来后可以部署cephfs的存储类
[root@master ~]# kubectl get po -n cephfs
NAME                                  READY   STATUS    RESTARTS   AGE
cephfs-provisioner-5dbd449769-sppgk   1/1     Running   0          3m45s


# 创建ceph的密钥
[root@master cephfsdemo]# cat ceph-secret.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: ceph-admin-secret
  namespace: cephfs
type: "kubernetes.io/rbd"
data:
  # ceph auth get-key client.admin |base64
  key: QVFCRitmUmM1c1FuR3hBQUtBeFl5cFNJaW4wbHFoQVh6NlRvQ2c9PQ==
[root@master cephfsdemo]# kubectl get secret -n cephfs
NAME                                                                       TYPE                                  DATA   AGE
ceph-admin-secret                                                          kubernetes.io/rbd                     1      94s
ceph-kubernetes-dynamic-user-c3f1789f-a4a0-11ea-b7d7-d61a8e0f81b2-secret   Opaque                                1      51s
cephfs-provisioner-token-p7nl4                                             kubernetes.io/service-account-token   3      43m
default-token-w6p8s                                                        kubernetes.io/service-account-token   3      43m


[root@master deploy]# cat cephfs-sc.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: cephfs
  namespace: kube-system
provisioner: ceph.com/cephfs
parameters:
    monitors: 10.234.2.204:6789			# 配置为ceph的mon地址
    adminId: admin
    adminSecretName: ceph-admin-secret		# 配置为ceph的admin 的secret
    adminSecretNamespace: "cephfs"
    
[root@master cephfsdemo]# kubectl apply -f cephfs-sc.yaml 
storageclass.storage.k8s.io/cephfs created
[root@master cephfsdemo]# kubectl get sc
NAME       PROVISIONER       AGE
ceph-rdb   ceph.com/rbd      174m
cephfs     ceph.com/cephfs   6s

  
# 创建pvc

[root@master cephfsdemo]# cat cephfs-pvc.yaml 
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: claim
  annotations:
    volume.beta.kubernetes.io/storage-class: "cephfs"
spec:
  accessModes:
    - ReadWriteMany		# 定义访问模式为多重读写
  resources:
    requests:
      storage: 1Gi
      
      
[root@master cephfsdemo]# kubectl get pvc
NAME    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
claim   Bound    pvc-c43afcb6-a4a0-11ea-b644-facf8ddba000   1Gi        RWX            cephfs         2m28s

# 创建用于测试的cephfs的pod
[root@master cephfsdemo]# cat cephfs-pod.yaml 
kind: Pod
apiVersion: v1
metadata:
  name: test-pod
spec:
  nodeSelector: 
    kubernetes.io/hostname: node1		# 使用nodeselect指定运行在node1上面
  containers:
  - name: test-pod
    image: nginx:latest
    volumeMounts:
      - name: pvc
        mountPath: "/mnt"						# 挂载目录为/mnt
  restartPolicy: "Never"
  volumes:
    - name: pvc
      persistentVolumeClaim:
        claimName: claim

[root@master cephfsdemo]# cat cephfs-pod2.yaml 
kind: Pod
apiVersion: v1
metadata:
  name: test-pod2
spec:
  nodeSelector: 
    kubernetes.io/hostname: node2		# 指定运行在node2上
  containers:
  - name: test-pod2
    image: nginx:latest
    volumeMounts:
      - name: pvc
        mountPath: "/data"				# 挂载目录/data
  restartPolicy: "Never"
  volumes:
    - name: pvc
      persistentVolumeClaim:
        claimName: claim
        
# 运行容器后
[root@master cephfsdemo]# kubectl get pod -owide
NAME        READY   STATUS    RESTARTS   AGE     IP             NODE    NOMINATED NODE   READINESS GATES
test-pod    1/1     Running   1          8m41s   10.244.1.87    node1   <none>           <none>
test-pod2   1/1     Running   0          16s     10.244.2.125   node2   <none>           <none>

# 查看挂载目录
[root@master cephfsdemo]# kubectl exec -it test-pod -- mount |grep /mnt
10.234.2.204:6789:/volumes/kubernetes/kubernetes/kubernetes-dynamic-pvc-c3f177f7-a4a0-11ea-b7d7-d61a8e0f81b2 on /mnt type ceph (rw,relatime,name=kubernetes-dynamic-user-c3f1789f-a4a0-11ea-b7d7-d61a8e0f81b2,secret=<hidden>,acl)
[root@master cephfsdemo]# kubectl exec -it test-pod2 -- mount |grep /data
10.234.2.204:6789:/volumes/kubernetes/kubernetes/kubernetes-dynamic-pvc-c3f177f7-a4a0-11ea-b7d7-d61a8e0f81b2 on /data type ceph (rw,relatime,name=kubernetes-dynamic-user-c3f1789f-a4a0-11ea-b7d7-d61a8e0f81b2,secret=<hidden>,acl)

# 进入到pod中测试
[root@master ~]# kubectl exec -it test-pod -- sh
# cd /mnt
# ls
# lls
sh: 5: lls: not found
# touch 1
touch: cannot touch '1': Input/output error

```

但是发现无法正常挂载，需要linux宿主机kernel版本4.10以上，可参考：https://github.com/kubernetes-incubator/external-storage/issues/345

## 5.4 升级系统kernel

```shell
[root@master updatakernel]# rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
[root@master updatakernel]# rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
Retrieving http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
Retrieving http://elrepo.org/elrepo-release-7.0-4.el7.elrepo.noarch.rpm
Preparing...                          ################################# [100%]
Updating / installing...
   1:elrepo-release-7.0-4.el7.elrepo  ################################# [100%]
[root@master updatakernel]# yum --disablerepo="*" --enablerepo="elrepo-kernel" list available

[root@master updatakernel]# yum --enablerepo=elrepo-kernel install kernel-ml
# 配置grub新增
[root@master ~]# cat /etc/default/grub 
GRUB_DEFAULT=0

# 生成grub
[root@master ~]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-4.4.225-1.el7.elrepo.x86_64
Found initrd image: /boot/initramfs-4.4.225-1.el7.elrepo.x86_64.img
Found linux image: /boot/vmlinuz-3.10.0-514.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-514.el7.x86_64.img
Found linux image: /boot/vmlinuz-0-rescue-689a0c92792f4df1aed088d246dfaaba
Found initrd image: /boot/initramfs-0-rescue-689a0c92792f4df1aed088d246dfaaba.img
done

# 注释/boot/grub2/grub.cfg 中老的grub信息

[root@master ~]# uname -rs
Linux 5.7.0-1.el7.elrepo.x86_64	
[root@master ~]# 
```







# 参考链接

* [ceph官方文档](http://docs.ceph.org.cn/) 
* [ceph中文开源社区]( http://ceph.org.cn/)
* [CentOS 7部署 Ceph分布式存储架构](https://www.cnblogs.com/happy1983/p/9246379.html)
* http://www.voidcn.com/article/p-cizuiqdb-bxp.html

* https://github.com/kubernetes-incubator/external-storage/issues/345
* https://www.jianshu.com/p/726bd9f37220