# TAPD 集成 GitLab

# 一 背景

## 1.1 需求

开发人员提交的代码无法自动和tapd上的story或task相关联，需要人工强制将提交的合并请求与tapd上面的ID关联，且无法无法形成有效记录和

需要tapd与gitlab联动，tapd提供源码提交趋势，并自动关联提交的。

在Gitlab CI完成后配置消息通知，检测服务器异常，已经pipeline发布状态，进行通知。

## 1.2 场景

* gitlab 提交代码带上commit号，在tapd上关联图形化展示
* tapd流水线集成企业使用持续集成平台，将持续集成平台数据在tapd流水线可视化。
* 通过企业微信机器人对流水线进行通知。

# 二 Gitlab配置

## 2.1 获取webhook信息

* 在Tapd的项目中的应用配置->添加应用->Gitlab源码

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211007231620.png)



* 开启Secret Token，获取到webhook url和项目token

DevOPS项目webhook:

```yaml
Webhook URL:https://www.tapd.cn/hook/index/67659124/6930c28927370ecfb819991d0cc2c730
Secret Token:af07d39ff1f8cb90660bb3799e840c03
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211007231721.png)

当gitlab 提交代码的时候可以通过POST请求通知到TAPD中。

## 2.2 关联Gitlab CI

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211007231747.png)

* 在Gitlab 项目中集成tapd

在项目->设置->集成中新增webhook。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211007231806.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211007231840.png)



# 三 Tapd配置流水线

## 3.1 代码关联

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211007231917.png)

## 3.2 持续集成关联

关联工具平台，TAPD支持关联Jenkins、Gitlab CI等工具平台。通过与TAPD进行联系，可以在TAPD查看指定构建任务的范围、结果与构建历史。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211007231946.png)

## 3.3 代码质量分析

TAPD支持关联主流代码检测工具。可以将团队正在使用的检测工具与TAPD进行关联，通过TAPD流水线查看每次代码扫描结果，进行问题跟踪与解决。

代码质量检测依赖于已关联的持续集成服务。以Jenkins为例，在Jenkins中安装SonarQube服务及插件，在已关联的构建任务中配置SonarQube步骤，即可通过TAPD流水线查看每次质量分析数据。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211007232011.png)

## 3.4 制品管理

制品/包管理依赖于已关联的持续集成服务。以Jenkins为例，在Jenkins中安装Nexus插件，通过在构建任务中添加Nexus Repository Manager Publisher 步骤，即可将每次构建制品信息展示在TAPD流水线中。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211007232045.png)

## 3.5 自动化测试

自动化测试执行依赖于已关联的持续集成服务。以Jenkins为例，在Jenkins 构建任务中配置自动化测试步骤，并在构建后步骤中添加“TAPD自动化测试报告解析”任务，填写自动化测试结果文件路径，即可在TAPD流水线中查看用例执行情况及测试报告。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211007232112.png)

## 3.6 发布部署

部署发布依赖于已关联的持续集成服务。以Jenkins为例，在Jenkins中安装Ansible插件，通过在构建任务中添加 Invoke Ansible Playbook步骤，即可在TAPD流水线中查看部署发布信息。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211007232200.png)



集成完成后，后续就可以通过gitlab 与tapd进行联动，从tapd上跳转到gitlab对应的代码提交页面。



注意：在提交代码的时候规范：

```shell
# 先写自己修改的内容：
fix: fix src_info
---
# 复制story中的源码提交关键字
--story=1008370 --user=薛磊 【示例】父需求 https://www.tapd.cn/48151165/s/1014039
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211007232249.png)

在gitlab 提交代码，在tapd查看流水线详情：

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211007232329.png)

![image-20211007232439768](/Users/xuel/Library/Application Support/typora-user-images/image-20211007232439768.png)

# 四 gitlab CI通过企业微信机器人通知

## 4.1 获取企业微信机器人webhook

可以在最后一步通过企业微信机器人通知构建消息。

可以先在企业微信拉取研发人员群，利用手机操作在群内拉去机器人，之后在构建的时候通过机器人通知。

机器人的webhook如下：

```shell
https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=777c9eaa-df67-4c69-9b0b-06394c067f5c
```

## 4.2 gitlab 配置机器人webhook

为保障微信机器人webhook 的安全性，将其配置在组的环境变量中，方便在ci中进行引用。

## 4.3 在gitlab ci中集成通知

由于使用了gitlab 共享模版库，集成非常方便，只需在gitlabci-templates的check job中集成即可

```shell
.check:
  script:
    - echo -e "\033[5;35;40m check ${APP_NAME} in k8s ${DEPLOY_ENV} environment status... \033[0m"
    - kubectl rollout status -w --timeout=300s deployment/${APP_NAME}-deploy -n ${NAMESPACE} &&
      curl ${WECHAT_WEBHOOK} -H 'Content-Type:application/json' -d "{\"msgtype\":\"markdown\",\"markdown\":{\"content\":\"${PROJECT_NAME}项目构建结果：<font color=\\"info\\">成功</font>\n>本次构建由：$GITLAB_USER_NAME 触发\n>项目名称：$CI_PROJECT_NAME\n>提交号：$CI_COMMIT_SHA\n>提交日志：$CI_COMMIT_MESSAGE\n>构建分支：$CI_COMMIT_REF_NAME\n>镜像信息：${REGISTRY}/${NAMESPACE}/${APP_NAME}:${APP_TAG}\n>流水线地址：[$CI_PIPELINE_URL]($CI_PIPELINE_URL)\"}}" ||
      curl ${WECHAT_WEBHOOK} -H 'Content-Type:application/json' -d "{\"msgtype\":\"markdown\",\"markdown\":{\"content\":\"${PROJECT_NAME}项目构建结果：<font color=\\"warning\\">失败</font>\n>本次构建由：$GITLAB_USER_NAME 触发\n>项目名称：$CI_PROJECT_NAME\n>提交号：$CI_COMMIT_SHA\n>提交日志：$CI_COMMIT_MESSAGE\n>构建分支：$CI_COMMIT_REF_NAME\n>镜像信息：${REGISTRY}/${NAMESPACE}/${APP_NAME}:${APP_TAG}\n>流水线地址：[$CI_PIPELINE_URL]($CI_PIPELINE_URL)\"}}"

  retry:
    max: 1
    when:
      - always
  allow_failure: true
```

![image-20201218112801316](/Users/xuel/Library/Application Support/typora-user-images/image-20201218112801316.png)

# 五 注意事项

* 将tapd上面源码集成中生成的webhook配置到gitlab时，不能在gitlab的组中集成中配置，需要在涉及到单个项目中配置。
* tapd 任务与gitlab 联动，需要在提交代码的时候复制源码提交关键字。

# 参考资料

* https://work.weixin.qq.com/api/doc/90000/90136/91770
* http://172.16.100.3:88/help/ci/variables/README#variables