## git常用操作

## 一 基本概念

- **工作区：**就是你在电脑里能看到的目录。
- **暂存区：**英文叫 stage 或 index。一般存放在 **.git** 目录下的 index 文件（.git/index）中，所以我们把暂存区有时也叫作索引（index）。
- **版本库：**工作区有一个隐藏目录 **.git**，这个不算工作区，而是 Git 的版本库。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220313155836.png)

## 二 基本操作

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220313160014.png)

- workspace：工作区
- staging area：暂存区/缓存区
- local repository：版本库或本地仓库
- remote repository：远程仓库

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220313161150.png)

1.已提交，但没push

* git reset —soft "撤销至上一次的id" ： 撤销commit，但是文件还是add的状态
* git reset —mixed "撤销到上一次的id"：撤销commit和add两个动作

2.以提交，并push

* git reset —hard "撤销到上一次的id"：撤销并舍弃版本号之后的提交记录，使用需谨慎，修改是在本地，需要推送到远程push -f，非常危险
* git revert "撤销那个id就写那个id"：撤销。但是保留提交记录。





## 参考链接

* https://github.com/LinXueyuanStdio/learnGit#%E9%9C%80%E6%B1%822%E5%8F%82%E4%B8%8Egithub%E5%BC%80%E6%BA%90%E9%A1%B9%E7%9B%AE%E6%8F%90%E4%BA%A4pull-request--back-to-%E7%9B%AE%E5%BD%95