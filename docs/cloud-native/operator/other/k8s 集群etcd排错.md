# k8s 集群etcd排错



![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210615174618.png)



查看etcd的pv被占用

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210615174703.png)



```shell
MountVolume.WaitForAttach failed for volume "pvc-03fd17c7-f766-4629-accf-49f2e4857b07" : rbd image kube/kubernetes-dynamic-pvc-14e0ccd9-83b0-4bd8-af16-0a200d356c8d is still being used
```



在node04 查看

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210615174757.png)



强制卸载



![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210615180859.png)



rbd unmap -o force /dev/rbd0



![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210615181043.png)





— 

重置网络

```
#在master节点之外的节点进行操作
kubeadm reset
systemctl stop kubelet
systemctl stop docker
rm -rf /var/lib/cni/
rm -rf /var/lib/kubelet/*
rm -rf /etc/cni/
ifconfig cni0 down
ipvsadm --clear 
ifconfig flannel.1 down
ifconfig docker0 down
ip link delete cni0
ip link delete flannel.1
##重启kubelet
systemctl restart kubelet
##重启docker
systemctl restart docker

iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat && ip link del kube-ipvs0 && ip link del nodelocaldns

modprobe -r ipip
lsmod
```



删除tunl



```shell
Linux删除tunnel的方法
测试lvs的时候，添加了一个IP tunnel，如下：

/sbin/ifconfig tunl0 $VIP broadcast $VIP netmask 255.255.255.255 up

/sbin/route add -host $VIP dev tunl0

测试完毕后，删除该IP tunnel时，死活删不掉，提示“ioctl：operation not permitted”，当然我是用root权限。

下面是我的一些尝试过程：
/sbin/ifconfig -a
―-晕，还是看到了tunl0好端端的在那里

/sbin/ifconfig tunl0 down
―-ping VIP,仍然ping的通

ip tunnel del tunl0
―-提示ioctl：operation not permitted

ip tunnel show
―-还是看到了tunl0好端端的在那里

google了下别人遇到这种问题怎么解决吧，晕，没有一个可行的。
―――――-华丽的救命线―――――――
最后啊，看了下我加载了哪些模块吧
lsmod
―-嗯，看到了模块ipip
好，卸载该模块
modprobe -r ipip
lsmod
―-不错，ipip被卸载了
ip tunnel show
―-哇，竟然tunl0不见了
/sbin/ifconfig -a
―-tunl0也看不到了
ping VIP
―-再也ping不通了
OK 到此为止
```



calico 异常；https://stackoverflow.com/questions/54465963/calico-node-is-not-ready-bird-is-not-ready-bgp-not-established

— 

https://www.cnblogs.com/00986014w/p/13683909.html

https://cloud.tencent.com/developer/article/1469532?from=information.detail.rbd:%20unmap%20failed:%20(16)%20device%20or%20resource%20busy



https://cloud.tencent.com/developer/article/1469533?from=article.detail.1469532