## 一、背景
我们工作中常用Jenkins部署Java代码，因其灵活的插件特性，例如jdk，maven，ant等使得java项目编译后上线部署一气呵成，同样对于脚本语言类型如Python上线部署，利用Jenkins强大的插件功能，轻松实现CI/CD，但如果部署多项目到同一台服务器涉及环境一致性问题，对此可以利用容器技术Docker解决，也可以利用Python虚拟环境例如virutalenv或conda等优秀等工具解决，在此由于后期根据requirements来安装依赖包比较慢，且后期需要将Python整个环境打包，利用conda工具来对项目环境进行管理，方便快速移植。
本文较系统的记录下部署一个具体的Django项目，包括利用conda工具来实现Python多环境管理，Pylint工具来实现代码检查，使用nose框架做单元测试和覆盖率。
## 二、必备知识
* Jenkins基础安装部署

可参考：[https://blog.51cto.com/kaliarch/2050862](https://blog.51cto.com/kaliarch/2050862)
* Conda软件包管理系统

由于conda包较大，通常情况下可以使用Miniconda
官网下载地址：[https://conda.io/en/latest/miniconda.html](https://conda.io/en/latest/miniconda.html)
基础使用命令：
```
一、工具包管理命令
1.更新工具包
conda update conda
conda upgrade --all

2.安装包(进入虚拟环境，也可用pip安装)
conda install package_name
可以指定版本
conda install package_name=1.10

3.移除包
conda remove package_name

4.查看包
conda list

5.查询包
conda search package_name

6.源配置
因为anaconda的服务器在国外，因此有时候速度会比较慢，可以换到国内源，比如清华的TUNA。

conda config --show-sources #查看当前使用源
conda config --remove channels 源名称或链接 #删除指定源
conda config --add channels 源名称或链接 #添加指定源

# 添加Anaconda的TUNA镜像
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
# TUNA的help中镜像地址加有引号，需要去掉
 
# 设置搜索时显示通道地址
conda config --set show_channel_urls yes


二、Python环境管理命令
1.显示创建的所有环境
conda env list

2.创建python环境：
conda create -n env_name list of packages
eg:conda create -n py2 python=2.7 request

3.进入python环境：(windows 环境不需要加source)
source activate env_name
eg:source activate py2

4.退出环境：
source deactivate

5.查看某环境下已安装的包
conda list -n python2

6.复制环境：
conda create --name <new_env_name> --clone <copied_env_name>

7.删除环境：
conda remove -n go2cloud-api-env --all

8.环境生成为YAML文件
当分享代码的时候，同时也需要将运行环境分享给大家，首先进入到环境中，执行如下命令可以将当前环境下的 package 信息存入名为 environment 的 YAML 文件中。 	
```
* conda env export > environment.yaml

同样，当执行他人的代码时，也需要配置相应的环境。这时你可以用对方分享的 YAML 文件来创建一摸一样的运行环境	

conda env create -f environment.yaml

Shell基础

可参考：[https://myshell-note.readthedocs.io/en/latest/index.html](https://myshell-note.readthedocs.io/en/latest/index.html)
* GIT基础

可参考：[https://blog.51cto.com/kaliarch/2049724](https://blog.51cto.com/kaliarch/2049724)
## 三、优化
* 生成主题

主题工具生成链接：[http://afonsof.com/jenkins-material-theme/](http://afonsof.com/jenkins-material-theme/)
在主题工具生成喜欢的颜色已经上传logo下载生成的主题到Jenkins服务器的jenkins 家目录，一般为安装启动jenkins系统用户的家目录下.jenkins/userContent/material/,如果没有此目录需要新建目录，css文件移动到目录下，例如/root/.jenkins/userContent/material/blue.css。
* Jenkins上配置主题

a.Install [Jenkins Simple Theme Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Simple+Theme+Plugin) 
b.点击Manage Jenkins 
c.点击Configure System 
d.找到 Theme 配置 
e.填写本地主题cssurl，例如：[http://localhost:8080/jenkins/userContent/material/blue.css](http://localhost:8080/jenkins/userContent/material/jenkins-material-theme-green.css) 
f.Click Save
g.保存可查看效果。
* 注意

本地css的url可以浏览器打开测试访问，如果访问不到会默认加载Jenkins默认主题
后期迁移url最好写成localhost，如果写公网IP，css文件不存在404，Jenkins页面会很卡。
## 四、部署实战
### 4.1 服务器列表
| 名称   | IP   | 软件   | 备注   | 
|:----:|:----:|:----:|:----:|
| Jenkins-server   | 10.57.61.138   | miniconda   | Jenkins服务器   | 
| Des-server   | 172.21.0.10   | miniconda   | 项目部署服务器   | 

### 4.2 架构图
![图片](https://uploader.shimo.im/f/HjVTJgoks7UUEdZ2.png!thumbnail)
### 4.3 前期准备
1. 安装依赖包
  * pylint: Python静态代码审查包，参考：[http://pylint.pycqa.org/en/latest/user_guide/run.html](http://pylint.pycqa.org/en/latest/user_guide/run.html)
  * mock: 用来生成测试数据。
  * nose: Python单元测试包。
  * coverage: Python代码覆盖率包。

对依赖包需要在Jenkins-server服务器进行安装，首先根据项目里面conda创建对应项目对虚拟环境`conda create -n <project_name> python=3.6`，创建完成利用`conda env list`查看环境，为避免环境污染，在项目环境内利用pip工具安装软件`pip install pylint mock nose coverage`。
1. 安装Jenkins插件
  * JUnit: 用来展示nose框架生成的单元测试报表(Allows JUnit-format test results to be published.)
  * Cobertura Plugin：用来展示Python代码测试覆盖率报表(This plugin integrates Cobertura coverage reports to Jenkins.)
  * Violations plugin：用来展示Python静态代码审查报表(This plugin does reports on checkstyle, csslint, pmd, cpd, fxcop, pylint, jcReport, findbugs, and perlcritic violations.)，参考：[https://wiki.jenkins.io/display/JENKINS/Violations](https://wiki.jenkins.io/display/JENKINS/Violations)
  * Git Plugin：用来从Gitbucket源代码库拉取代码(This plugin allows GitLab to trigger Jenkins builds and display their results in the GitLab UI.)
  * Git Parameter：用于参数化构建选择git的branch（Adds ability to choose branches, tags or revisions from git repositories configured in project.）
### 4.4 创建任务
* 创建自由风格软件项目

New任务->构建一个自由风格的软件项目,填写描述，在此由于后期会利用pylint进行代码检查，给出了代码检查的消息类型，可以根据消息类型进行相应修复处理。
![图片](https://uploader.shimo.im/f/n38do1PSGowT2sbr.png!thumbnail)
* 配置参数化构建

配置选择git具体branch进行构建，和可以自定义端口，在此需要注意参数化构建的变了Name，在后续需要用到。
![图片](https://uploader.shimo.im/f/kx55lH4kG5oQi5TG.png!thumbnail)
* 源码管理

在此需要选择源码仓库，选择gitlab已经认证方式，需要注意由于参数化构建选择了branch，在Branches to build需要引用上面的变量$branch。
![图片](https://uploader.shimo.im/f/FXvZZilvmN0ByL1p.png!thumbnail)
* 构建配置
  * 执行shell

执行shell此shell为在Jenkins服务器上执行，所以需要预先在其上配置Python虚拟环境，在其上进行代码审查，单元测试生成nosetests.xml文件，已经代码覆盖率测试生成coverage.xml文件，pylint测试生成pylint.xml文件。
![图片](https://uploader.shimo.im/f/TpUiVStCLzcJ5cy4.png!thumbnail)

以下为此项目示例shell脚本，此脚本需要根据自己的实际情况来修改，需要注意Python项目结构与需要代码检查的标识符。
```
base_dir=/root/.jenkins/workspace/
project=go2cloud-api-deploy-prod/
project_env=go2cloud-api-env
# 切换python环境
source activate ${project_env}
$(which python) -m pip install mock nose coverage
# 更新python环境
echo "+++更新Python环境+++"

if [ -f ${base_dir}${project}requirements.txt ];then
    $(which python) -m pip install -r ${base_dir}${project}requirements.txt && echo 0 || echo 0
fi

# 代码检查/单元测试/代码测试覆盖率
echo "+++代码检查+++"
cd ${base_dir}
# 生成pylint.xml
$(which pylint) -f parseable --disable=C0103,E0401,C0302 $(find ${project}/* -name *.py) >${base_dir}pylint.xml || echo 0

echo "+++单元测试+++"
# 生成nosetests.xml
#$(which nosetests) --with-xunit --all-modules --traverse-namespace --with-coverage --cover-package=go2cloud-api-deploy-prod --cover-inclusive || echo 0
$(which nosetests) --with-xunit --all-modules --traverse-namespace --with-coverage --py3where=go2cloud-api-deploy-prod --cover-package=go2cloud-api-deploy-prod --cover-inclusive || echo 0
echo "+++代码覆盖率+++"
# 生成coverage.xml
```
python -m coverage xml --include=go2cloud-api-deploy-prod* || echo 0


  * 发送文件及命令到目标服务器

![图片](https://uploader.shimo.im/f/GAlRFfzeaJEQbiof.png!thumbnail)
在此需要制定发布到目标到的服务器远端目录，已经执行的命令，在此执行重启脚本。
由于在此项目为Django项目，需要制定虚拟环境/入口启动文件/启动端口。
将之前参数化构建的端口当作变量传递给脚本，启动相应的端口
```
#!/usr/bin/env bash

# 当前目录
BASEPATH=$(cd `dirname $0`;pwd)

# python解释器具体路径
PYTHON_BIN=$1

# mananger文件路径
MAIN_APP=$2

# python
SERVER_PORT=$3

[ $# -lt 3 ] && echo "缺少参数" && exit 1

LOG_DIR=${BASEPATH}/logs/
[ ! -d ${LOG_DIR} ] && mkdir ${LOG_DIR}

OLD_PID=`netstat -lntup | awk -v SERVER_PORT=${SERVER_PORT} '{if($4=="0.0.0.0:"SERVER_PORT) print $NF}'|cut -d/ -f1`
[ -n "${OLD_PID}" ] && kill -9 ${OLD_PID}

echo "---------$0 $(date) excute----------" >> ${LOG_DIR}server-$(date +%F).log

# 启动服务
```
nohup ${PYTHON_BIN} -u ${MAIN_APP} runserver 0.0.0.0:${SERVER_PORT} &>> ${LOG_DIR}server-$(date +%F).log 2>&1 &


* 构建后动作
  * JUnit插件实现单元测试报告，需要指定nosetests.xml

![图片](https://uploader.shimo.im/f/dV9ndseOrxoNbKpx.png!thumbnail)
  * Cobertura Plugin插件实现覆盖率测试

![图片](https://uploader.shimo.im/f/18SBxO4Qks4ZjyKl.png!thumbnail)
  * Violations插件进行代码审计，需要制定Jenkins-server上的生成的pylint.xml文件。

需要注意文件路径为jenkins服务器pylint.xml，以及对应生成文件的编码。
![图片](https://uploader.shimo.im/f/k6fYgkdPPm80ivcD.png!thumbnail)
![图片](https://uploader.shimo.im/f/7r0RooWT1lIDGGSi.png!thumbnail)

  * 邮件通知配置

选择邮件内容为Content Type为HTML，这样可以编写邮件HTML模版，生成较为好看的邮件通知模版。
注意选择触发告警可以选择类型，失败几次或无论构建成功失败都发送，可根据具体需求配置。
![图片](https://uploader.shimo.im/f/12scxyG3dNYHdEWn.png!thumbnail)
![图片](https://uploader.shimo.im/f/JkP2KV5IgSIbHUnn.png!thumbnail)
邮件HTML模版
```
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>${ENV, var="JOB_NAME"}-第${BUILD_NUMBER}次构建日志</title>
</head>

<body leftmargin="8" marginwidth="0" topmargin="8" marginheight="4"
    offset="0">
    <table width="95%" cellpadding="0" cellspacing="0"
        style="font-size: 11pt; font-family: Tahoma, Arial, Helvetica, sans-serif">
        <tr>
            <td>(本邮件是程序自动下发的，请勿回复！)</td>
        </tr>
        <tr>
            <td><h2>
                    <font color="#0000FF">构建结果 - ${BUILD_STATUS}</font>
                </h2></td>
        </tr>
        <tr>
            <td><br />
            <b><font color="#0B610B">构建信息</font></b>
            <hr size="2" width="100%" align="center" /></td>
        </tr>
        <tr>
            <td>
                <ul>
                    <li>项目名称&nbsp;：&nbsp;${PROJECT_NAME}</li>
                    <li>构建编号&nbsp;：&nbsp;第${BUILD_NUMBER}次构建</li>
                    <li>SVN&nbsp;版本：&nbsp;${SVN_REVISION}</li>
                    <li>触发原因：&nbsp;${CAUSE}</li>
                    <li>构建日志：&nbsp;<a href="${BUILD_URL}console">${BUILD_URL}console</a></li>
                    <li>构建&nbsp;&nbsp;Url&nbsp;：&nbsp;<a href="${BUILD_URL}">${BUILD_URL}</a></li>
                    <li>工作目录&nbsp;：&nbsp;<a href="${PROJECT_URL}ws">${PROJECT_URL}ws</a></li>
                    <li>项目&nbsp;&nbsp;Url&nbsp;：&nbsp;<a href="${PROJECT_URL}">${PROJECT_URL}</a></li>
                </ul>
            </td>
        </tr>
        <tr>
            <td><b><font color="#0B610B">Changes Since Last
                        Successful Build:</font></b>
            <hr size="2" width="100%" align="center" /></td>
        </tr>
        <tr>
            <td>
                <ul>
                    <li>历史变更记录 : <a href="${PROJECT_URL}changes">${PROJECT_URL}changes</a></li>
                </ul> ${CHANGES_SINCE_LAST_SUCCESS,reverse=true, format="Changes for Build #%n:<br />%c<br />",showPaths=true,changesFormat="<pre>[%a]<br />%m</pre>",pathFormat="&nbsp;&nbsp;&nbsp;&nbsp;%p"}
            </td>
        </tr>
        <tr>
            <td><b>Failed Test Results</b>
            <hr size="2" width="100%" align="center" /></td>
        </tr>
        <tr>
            <td><pre
                    style="font-size: 11pt; font-family: Tahoma, Arial, Helvetica, sans-serif">$FAILED_TESTS</pre>
                <br /></td>
        </tr>
        <tr>
            <td><b><font color="#0B610B">构建日志 (最后 100行):</font></b>
            <hr size="2" width="100%" align="center" /></td>
        </tr>
        <!-- <tr>
            <td>Test Logs (if test has ran): <a
                href="${PROJECT_URL}ws/TestResult/archive_logs/Log-Build-${BUILD_NUMBER}.zip">${PROJECT_URL}/ws/TestResult/archive_logs/Log-Build-${BUILD_NUMBER}.zip</a>
                <br />
            <br />
            </td>
        </tr> -->
        <tr>
            <td><textarea cols="80" rows="30" readonly="readonly"
                    style="font-family: Courier New">${BUILD_LOG, maxLines=100}</textarea>
            </td>
        </tr>
    </table>
</body>
</html>
```

### 4.5 进行构建测试
* 触发构建

选择对应的branch与端口
![图片](https://uploader.shimo.im/f/eXFdYmyypTEOBoZU.png!thumbnail)
* 查看consle log
  * pylint检查

![图片](https://uploader.shimo.im/f/6ZSFsNT7Z78OXB4A.png!thumbnail)
  * nosetests单元测试及代码覆盖率

![图片](https://uploader.shimo.im/f/iQ6nEH8yahEgEkYH.png!thumbnail)
![图片](https://uploader.shimo.im/f/AKV7fv94fXwzGkIl.png!thumbnail)
  * 目标服务器查看具体项目

![图片](https://uploader.shimo.im/f/lqjzBnVlZqMRcLIv.png!thumbnail)

### 4.6 查看构结果
* 总览查看

![图片](https://uploader.shimo.im/f/R2TTW3Qft0gAJgrh.png!thumbnail)
* 查看代码覆盖率

![图片](https://uploader.shimo.im/f/0xg6TUUDGhQr3WSO.png!thumbnail)
查看具体不合规代码
在源代码分析结束后面，会有一系列的报告，每个报告关注于项目的某些方面，如每种类别的 message 的数目，模块的依赖关系等等。具体来说，报告中会包含如下的方面：
MESSAGE_TYPE 有如下几种：
```
(C) convention 惯例。违反了编码风格标准
(R) refactor 重构。写得非常糟糕的代码。
(W) warning 警告。某些 Python 特定的问题。
(E) error 错误。很可能是代码中的错误。
(F) fatal 致命错误。阻止 Pylint 进一步运行的错误。
```
![图片](https://uploader.shimo.im/f/MtsV8v6nMwQwWfIf.png!thumbnail)
* 查看邮件

![图片](https://uploader.shimo.im/f/GWDXN3IVKyQoGxaq.png!thumbnail)
## 五、流水线部署
### 5.1 pipeline基础概念
* pipeline是什么

pipeline为运行于Jenkins上的工作流框架，project中的配置信息以steps的方式放在一个脚本里将原本独立运行于单个或者多个节点的任务连接起来，实现单个任务难以完成的复杂流程编排与可视化。
* 基础概念
  * Stage: 阶段：告诉Jenkins做什么，一个Pipeline可以划分为若干个Stage，每个Stage代表一组操作。注意，Stage是一个逻辑分组的概念，可以跨多个Node。
  * Node: 节点：告诉Jenkins在job在哪运行一个Node就是一个Jenkins节点，或者是Master，或者是slave，是执行Step的具体运行期环境。
  * Step: 步骤：具体细化到每一步构建操作，Step是最基本的操作单元，小到创建一个目录，大到构建一个Docker镜像，由各类Jenkins Plugin提供。
* 语发工具

Pipeline提供了一组可扩展的工具，通过Pipeline Domain Specific Language（DSL）syntax可以达到Pipeline as Code（Jenkinsfile存储在项目的源代码库）的目的。

参考：[https://wiki.jenkins.io/display/JENKINS/Pipeline+Plugin](https://wiki.jenkins.io/display/JENKINS/Pipeline+Plugin)
### 5.2 pipeline的特性
基于 Jenkins Pipeline，用户可以在一个 JenkinsFile 中快速实现一个项目的从构建、测试以到发布的完整流程，并且可以保存这个流水线的定义。
* 代码:Pipeline以代码的形式实现，通常被检入源代码控制，使团队能够编辑、审查和迭代其CD流程。
* 可持续性：Jenklins重启或者中断后都不会影响Pipeline Job。
* 停顿：Pipeline可以选择停止并等待任工输入或批准，然后再继续Pipeline运行。
* 多功能：Pipeline支持现实世界的复杂CD要求，包括fork/join子进程，循环和并行执行工作的能力
* 可扩展：Pipeline插件支持其DSL的自定义扩展以及与其他插件集成的多个选项。
### 5.3 pipeline语法
* 声明式
```
pipeline {
/* insert Declarative Pipeline here */
}
```
在声明式流水线中有效的基本语句和表达式遵循与 [Groovy的语法](http://groovy-lang.org/syntax.html)同样的规则， 有以下例外:
* 流水线顶层必须是一个 *block*, 特别地: pipeline { }
* 没有分号作为语句分隔符，，每条语句都必须在自己的行上。
* 块只能由 [节段](https://jenkins.io/zh/doc/book/pipeline/syntax/#declarative-sections), [指令](https://jenkins.io/zh/doc/book/pipeline/syntax/#declarative-directives), [步骤](https://jenkins.io/zh/doc/book/pipeline/syntax/#declarative-steps), 或赋值语句组成。 *属性引用语句被视为无参方法调用。 例如, input被视为 input()

示例：
```
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent none 
    stages {
        stage('Example Build') {
            agent { docker 'maven:3-alpine' } 
            steps {
                echo 'Hello, Maven'
                sh 'mvn --version'
            }
        }
        stage('Example Test') {
            agent { docker 'openjdk:8-jre' } 
            steps {
                echo 'Hello, JDK'
                sh 'java -version'
            }
        }
    }
}
```
详细学习可参考：[https://jenkins.io/zh/doc/book/pipeline/syntax/](https://jenkins.io/zh/doc/book/pipeline/syntax/)
* 脚本式

脚本化流水线, 与[[declarative-pipeline]](https://jenkins.io/zh/doc/book/pipeline/syntax/#declarative-pipeline)一样的是, 是建立在底层流水线的子系统上的与声明式不同的是, 脚本化流水线实际上是由 [Groovy](http://groovy-lang.org/syntax.html)构建的通用 DSL [[2](https://jenkins.io/zh/doc/book/pipeline/syntax/#_footnotedef_2)]。 Groovy 语言提供的大部分功能都可以用于脚本化流水线的用户。这意味着它是一个非常有表现力和灵活的工具，可以通过它编写持续交付流水线。

示例：
```
node('master') {     //master节点运行，以下stage也可指定节点
    stage 'Prepare'  //清空发布目录
        bat '''if exist D:\\publish\\LoginServiceCore (rd/s/q D:\\publish\\LoginServiceCore)
               if exist C:\\Users\\Administrator\\.nuget (rd/s/q C:\\Users\\Administrator\\.nuget)
               exit'''

    //拉取git代码仓库
    stage 'Checkout'
        checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], 
　　　　　　　submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'c6d98bbd-5cfb-4e26-aa56-f70b054b350d', 
            url: 'http://xxx/xxx/xxx']]])
   
    //构建
    stage 'Build'
        bat '''cd "D:\\Program Files (x86)\\Jenkins\\workspace\\LoginServiceCore\\LoginApi.Hosting.Web"
            dotnet restore
            dotnet build
            dotnet publish --configuration Release --output D:\\publish\\LoginServiceCore'''
    
    //部署
    stage 'Deploy'
        bat '''
        cd D:\\PipelineScript\\LoginServiceCore
        python LoginServiceCore.py
        '''

    //自动化测试（python代码实现）    
    stage 'Test'
        bat'''
        cd D:\\PipelineScript\\LoginServiceCore
        python LoginServiceCoreApitest.py
        '''   
}
```

### 5.4 示例
在此将上面的项目利用pipeline进行发布
* 创建自由风格软件项目

New任务->流水线，填写任务描述。
![图片](https://uploader.shimo.im/f/NGOErGR5XSAO0STU.png!thumbnail)

![图片](https://uploader.shimo.im/f/z8CIqV6kUQ0vQlC3.png!thumbnail)

* Pipeline

如果对Pipeline语法不熟悉，可以利用工具生成
![图片](https://uploader.shimo.im/f/IkHAJ2Pv2qcqB5OS.png!thumbnail)
在此的pipeline
```
pipeline {
  agent any
  parameters {
    gitParameter branchFilter: 'origin/(.*)', defaultValue: 'master', name: 'BRANCH', type: 'PT_BRANCH'
  }
  stages {
    stage('checkout src code') {
        steps {
            echo "checkout src code"
            git branch: "${params.BRANCH}",'http://123.206.xxx.xxx/xuel/go2cloud_platform.git'
        }
    }
    stage('exec shell'){
        steps{
            echo "pylint,Unit test"
            sh '''# jenkins 服务器项目workspace目录
                base_dir=/root/.jenkins/workspace/
                # 项目名称
                project=go2cloud_platform
                # 项目环境python环境
                project_env=go2cloud_platform_pipeline
                # 切换python环境
                source /data/miniconda3/bin/activate ${project_env}
                $(which python) -m pip install mock nose coverage pylint
                # 更新python环境
                echo "++++++更新Python环境+++"
                
                if [ -f ${base_dir}${project}requirements/requirements.txt ];then
                    $(which python) -m pip install -r ${base_dir}${project}/requirements/requirements.txt && echo 0 || echo 0
                fi
                
                # 代码检查/单元测试/代码测试覆盖率
                echo "++代码检查++"
                cd ${base_dir}
                # 生成pylint.xml
                $(which pylint) -f parseable --disable=C0103,E0401,C0302 $(find ${project}/* -name *.py) >${base_dir}${project}_pylint.xml || echo 0
                
                echo "++单元测试++"
                # 生成nosetests.xml
                #$(which nosetests) --with-xunit --all-modules --traverse-namespace --with-coverage --cover-package=go2cloud-api-deploy-prod --cover-inclusive || echo 0
                $(which nosetests) --with-xunit --all-modules --traverse-namespace --with-coverage --py3where=${project} --cover-package=${project} --cover-inclusive || echo 0
                echo "++代码覆盖率+++"
                # 生成coverage.xml
                python -m coverage xml --include=${project_env}* || echo 0'''

        }
    }
    stage("deploy") {
        steps {
            echo "send file"
            sshPublisher(publishers: [sshPublisherDesc(configName: 'go2cloud_platform_host', transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: '''base_dir=/devops_pipeline
                project_src=go2cloud_platform
                project_env=/data/miniconda3/envs/go2cloud_platform_pipeline/bin/python
                echo "+++++++更新部署服务器python环境++++++++++"
                if [ -f ${base_dir}/requirements/requirements.txt ];then
                    ${project_env} -m pip install -r ${base_dir}/requirements/requirements.txt && echo 0 || echo 0
                fi
                echo -e "\\033[32m 启动服务脚本 \\033[0m"
                $(which bash) ${base_dir}/run_server.sh ${project_env} ${base_dir}/apps/manage.py ${port}
                ''', execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '/devops_pipeline', remoteDirectorySDF: false, removePrefix: '', sourceFiles: '**/**')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])
            }
        }
    }
}
```
* 运行构建

![图片](https://uploader.shimo.im/f/sccaijAd9pEUzCsj.png!thumbnail)
* 查看结果

![图片](https://uploader.shimo.im/f/ult7GreavjgCjKTC.png!thumbnail)
![图片](https://uploader.shimo.im/f/mxEJeCLV8E4K3tIi.png!thumbnail)
## 五、注意要点
* 在部署前一定梳理好流程，需要理解哪一步在目标服务器还上在Jenkins服务器上执行
* 需要对应好目录及Python虚拟环境，避免环境污染
## 六、反思
* 在此利用来Conda虚拟环境管理，来在同一个服务器上解决环境不一致行，也可以利用Docker来解决
* 利用Pipeline项目发布可视化，明确阶段，方便stage查看与排错，Jenkinsfile可放在git仓库进行

