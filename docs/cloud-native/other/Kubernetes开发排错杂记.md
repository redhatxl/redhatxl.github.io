## Kubernetes开发排错杂记

## 前言

在云原生开发中，回遇到很多比较细小的点，在此单独记录。

## 一 单个POD包含多个容器

在利用client-go开发K8s创建POD时候，一个POD包含多个容器，经过测试，Container列表与顺序有关，第一个container需要时主容器，要有一定的先后顺序。

![image-20210604110448227](/Users/xuel/Library/Application Support/typora-user-images/image-20210604110448227.png)

## 二 传递字段类型需要进行转换

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210604110544.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210604110557.png)



或者使用"k8s.io/utils/pointer"中的具体类型转指针

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210915122408.png)

## 三 创建pod查看没有挂载serviceaccount

需要在pod的spec指定`AutomountServiceAccountToken`为true

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210604110656.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210604110727.png)

## 四 TODO

