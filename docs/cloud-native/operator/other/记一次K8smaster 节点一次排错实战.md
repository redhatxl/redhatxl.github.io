# 记一次K8smaster 节点一次排错实战

## 一 背景

### 1.1 监控报警master01/master03有告警

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210603153857.png)

### 1.2 登陆查看api-server pod状态

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210603154037.png)

## 二 操作



### 2.1 查看pod信息

* 查看pod状态



![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210603154850.png)

* 查看日志

```shell
[root@master01 ssl]# kubectl logs -f -n kube-system kube-apiserver-master01
```



![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210603154557.png)

* 查看监控信息

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210603161701.png)



### 2.2 检测证书

不知道证书在哪里，可以查看apiserver服务使用的证书进行验证。

```shell
curl --cacert /etc/kubernetes/ssl/ca.crt --cert =/etc/kubernetes/ssl/apiserver-kubelet-client.crt --key /etc/kubernetes/ssl/apiserver-kubelet-client.key https://127.0.0.1:6443/healthz
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210603153758.png)

## 三 master内存太小

### 3.1 编写脚本

```shell
#!/bin/bash

#内存使用超过阀值
WARN_LINE=20
# 日志记录文件
LOG_DIR=/var/log/memfree/
[ ! -d ${LOG_DIR} ] && mkdir -p ${LOG_DIR}
LOG_FILE=$(date +%F)-memefree.log
LOG_TOTLE=${LOG_DIR}${LOG_FILE}

# 内存总量
MEM_TOTLE=$(free  -m | awk 'NR==2{print $2}')

# 内存使用量
#MEM_USE=$(free  -m | awk 'NR==2{print $3}')

# 内存空闲
MEM_FREE=$(free  -m | awk 'NR==2{print $4}')
# 已使用百分比
#USE_PERCENT=$(printf "%5f" `echo "scale=5;${MEM_USE}/${MEM_TOTLE}"|bc`)
USE_PERCENT=$(awk -v use=${MEM_FREE} -v totle=${MEM_TOTLE} 'BEGIN{printf "%0.0f",use/totle*100}')

echo ${USE_PERCENT}
if [[ ${USE_PERCENT} -le ${WARN_LINE} ]];then
        echo "---------$(date +%F" "%T) mem free begin---------" >> ${LOG_TOTLE}
        echo "内存释放前，使用情况如下:" >> ${LOG_TOTLE}
        free -m &>>${LOG_TOTLE}
        sync
        echo 1 > /proc/sys/vm/drop_caches
        echo 2 > /proc/sys/vm/drop_caches
        echo 3 > /proc/sys/vm/drop_caches
        echo "内存释放结束后，使用情况如下:" >> ${LOG_TOTLE}
        free -m &>>${LOG_TOTLE}
        echo "---------$(date +%F" "%T) mem free end---------" >> ${LOG_TOTLE}
fi
```

### 3.2 制作定时任务

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210603162208.png)

* 查看执行后结果

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210603162444.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210603162222.png)

## 四 反思

之后集群再也不会出现类似内存不足的情况。
