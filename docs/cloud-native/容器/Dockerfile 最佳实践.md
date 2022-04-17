# Dockerfile 最佳实践

## 一 减少构建时间

### 1.1 构建顺序影响缓存利用率

**把不需要经常更改的行放到最前面，更改最频繁的行放到最后面**

镜像的构建顺序很重要，当你向 Dockerfile 中添加文件，或者修改其中的某一行时，那一部分的缓存就会失效，该缓存的后续步骤都会中断，需要重新构建。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220404200747.png)

### 1.2 只拷贝需要文件，防止缓存溢出

当拷贝文件到镜像中时，尽量只拷贝需要的文件，切忌使用 COPY . 指令拷贝整个目录。如果被拷贝的文件内容发生了更改，缓存就会被破坏。在上面的示例中，镜像中只需要构建好的 jar 包，因此只需要拷贝这个文件就行了，这样即使其他不相关的文件发生了更改也不会影响缓存。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220404200759.png)

### 1.3 最小化可缓存的执行层

每一个RUN指令都会被看作可缓存的执行单元，太多的RUN指令会增加镜像的层数，增大镜像的体积，而将所有的指令都放在一个RUN指令又会破坏缓存，从而延缓开放周期，一般都会先更新软件索引信息，然后再安装软件。推荐将更新索引和安装软件放在同一个 RUN 指令中，这样可以形成一个可缓存的执行单元，否则你可能会安装旧的软件包。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220404200810.png)

### 1.4 删除比必要的依赖

删除不必要的依赖，不要安装调试工具。如果实在需要调试工具，可以在容器运行之后再安装。某些包管理工具（如 apt）除了安装用户指定的包之外，还会安装推荐的包，这会无缘无故增加镜像的体积。apt 可以通过添加参数 -–no-install-recommends 来确保不会安装不需要的依赖项。如果确实需要某些依赖项，请在后面手动添加。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220404200822.png)

### 1.5 删除管理工具的缓存

包管理工具会维护自己的缓存，这些缓存会保留在镜像文件中，推荐的处理方法是在每一个 RUN 指令的末尾删除缓存。如果你在下一条指令中删除缓存，不会减小镜像的体积。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220404200833.png)

## 二 可维护性

### 2.1 尽可能使用官方镜像

使用官方镜像可以节省大量的维护时间，因为官方镜像的所有安装步骤都使用了最佳实践。如果你有多个项目，可以共享这些镜像层，因为他们都可以使用相同的基础镜像。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220404200845.png)

### 2.2 使用更具体的标签

基础镜像尽量不要使用 latest 标签。虽然这很方便，但随着时间的推移，latest 镜像可能会发生重大变化。因此在 Dockerfile 中最好指定基础镜像的具体标签。我们使用 openjdk 作为示例，指定标签为 8。其他更多标签请查看官方仓库。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220404200858.png)

### 2.3 使用体积更小的基础镜像

基础镜像的标签风格不同，镜像体积就会不同。slim 风格的镜像是基于 Debian 发行版制作的，而 alpine 风格的镜像是基于体积更小的 Alpine Linux 发行版制作的。其中一个明显的区别是：Debian 使用的是 GNU 项目所实现的 C 语言标准库，而 Alpine 使用的是 Musl C 标准库，它被设计用来替代 GNU C 标准库（glibc）的替代品，用于嵌入式操作系统和移动设备。因此使用 Alpine 在某些情况下会遇到兼容性问题。以 openjdk 为例，jre 风格的镜像只包含 Java 运行时，不包含 SDK，这么做也可以大大减少镜像体积。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220404200913.png)

### 2.4 给镜像设置合适的LABEL

每一个对象都包含相应的元数据信息，比如 Docker 镜像就包含作者、大小等等元数据信息。在使用镜像的时候，往往会通过这些元数据信息来查找适合的镜像来用于开发或测试，而不单单只是通过名字去检索。

在 Dockerfile 中可以使用 Label 命令来为镜像增加 Label，示例如下：

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220404200923.png)

我们可以查看到 Label 及使用 Label 进行筛选:

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220404200936.png)

## 三 重复利用

### 3.1 在一致的环境中从源码构建

首先应该确定构建应用所需的所有依赖，本文的示例 Java 应用很简单，只需要 Maven 和 JDK，所以基础镜像应该选择官方的体积最小的 maven 镜像，该镜像也包含了 JDK。如果你需要安装更多依赖，可以在 RUN 指令中添加。pom.xml 文件和 src 文件夹需要被复制到镜像中，因为最后执行 mvn package 命令（-e 参数用来显示错误，-B 参数表示以非交互式的“批处理”模式运行）打包的时候会用到这些依赖文件。



虽然现在我们解决了环境不一致的问题，但还有另外一个问题：每次代码更改之后，都要重新获取一遍 pom.xml 中描述的所有依赖项。下面我们来解决这个问题。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220404200945.png)

### 3.2 在单独的步骤中获取依赖项目

结合前面提到的缓存机制，我们可以让获取依赖项这一步变成可缓存单元，只要 pom.xml 文件的内容没有变化，无论代码如何更改，都不会破坏这一层的缓存。上图中两个 COPY 指令中间的 RUN 指令用来告诉 Maven 只获取依赖项。

现在又遇到了一个新问题：跟之前直接拷贝 jar 包相比，镜像体积变得更大了，因为它包含了很多运行应用时不需要的构建依赖项。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220404201009.png)



### 3.3 使用多阶段构建来删除构建时的依赖

多阶段构建可以由多个 FROM 指令识别，每一个 FROM 语句表示一个新的构建阶段，阶段名称可以用 AS 参数指定。本例中指定第一阶段的名称为 builder，它可以被第二阶段直接引用。两个阶段环境一致，并且第一阶段包含所有构建依赖项。

第二阶段是构建最终镜像的最后阶段，它将包括应用运行时的所有必要条件，本例是基于 Alpine 的最小 JRE 镜像。上一个构建阶段虽然会有大量的缓存，但不会出现在第二阶段中。为了将构建好的 jar 包添加到最终的镜像中，可以使用 COPY --from=STAGE_NAME 指令，其中 STAGE_NAME 是上一构建阶段的名称。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220404201033.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220404201043.png)

## 四 构建

### 4.1 分阶段构建

构建镜像最大的挑战莫过于防止镜像过大，造成实际运行时由于并发而导致拉取性能问题。为了应对这个挑战，很多转型到容器的团队采用两个 Dockerfile，一个负责开发环境的镜像构建，一个负责生产环境的镜像构建。开发镜像包含了代码构建所需要的环境，镜像大小自然比较大，生产镜像仅包含应用运行所需要的内容，是很精简的体积很小的镜像。



Docker 分阶段构建就可以很好地解决这个问题，使用如下的 Dockerfile 即可：

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220404201055.png)

整个 Dockerfile 流程很清晰，也不在需要额外的 shell 脚本来支持整个流程，并且我们可以指定执行的 stage，具体命令如下：

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220404201105.png)