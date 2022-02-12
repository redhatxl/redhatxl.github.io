# 一 概述

对于项目代码质量管理，在目前的微服务/模块化/快迭代敏捷开发中如果仅依赖IDE简单检查和人为的codereview对于大量代码很不适合，不仅仅依靠开发人员的编码规范编码及注意程序健壮性，同时需要一些工具来帮助我们提前预防和强制检测规范。

Sonarqube 是一款代码分析检测工具，将其与devops结合，例如集成到gitlab ci/cd或jenkins中实现部署自动代码检查，及时发现并处理bug，最大限度的将bug和不规范扼杀在编码阶段，其内部集成很多分析工具，比如pmd-cpd、checkstyle、findbugs、Jenkins，从七个方面帮我们来源码质量管理。此文章安装最新版SonarQube-7.9.1，此版本不支持自定义数据库MySQL，jdk需要安装高版本11。

## 1.1 特点

* 检查代码是否遵循编程标准：如命名规范，编写的规范等。
* 检查设计存在的潜在缺陷：SonarQube 通过插件 Findbugs、Checkstyle 等工具检测代码存在的缺陷。
* 检测代码的重复代码量：SonarQube 可以展示项目中存在大量复制粘贴的代码。
* 检测代码中注释的程度：源码注释过多或者太少都不好，影响程序的可读可理解性。

* 检测代码中包、类之间的关系：分析类之间的关系是否合理，复杂度情况。

## 1.2 组件

- SonarQube Server：sonarqube服务端，接受客户端扫描报告
- SonarQube Database：ES/及数据库引擎oracle，postgresql，mssql
- SonarQube Plugins：可以后期在sonarqube服务端安装插件
- SonarQube Scanner：安装在客户端扫描工具

## 1.3 架构图

![image-20190810145350121](/Users/xuel/Library/Application Support/typora-user-images/image-20190810145350121.png)

开发人员把代码push到SCM（如gitlab）->jenkins构建定义好的job，然后通过jenkins 插件（sonar scanner）分析源码->jenkins/gitlab-ci 中的scanner客户端把分析报告发到sonarqube server

# 二 安装部署

## 2.1 SonarQube Server安装

### 2.1.1 宿主机测试环境简单部署

```shell
# 下载
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-7.9.1.zip
# 解压
/opt/sonarqube/
unzip /opt/sonarqube/bin/[OS]/sonar.sh console

# 登录宿主机http://localhost:9000 （admin/admin）
```

### 2.1.2 宿主机生产环境部署

```shell
# 宿主机requirementes
sysctl -w vm.max_map_count=262144
sysctl -w fs.file-max=65536
ulimit -n 65536
ulimit -u 4096

cat >> /etc/sysctl.conf  << EOF
vm.max_map_count=262144
fs.file-max=65536
EOF


# sonarqube不能用root用户执行
useradd sonarqube
echo "sonarqubepwd" | passwd --stdin sonarqube

# 检查系统
[root@devops-sonarqube ~]# grep SECCOMP /boot/config-$(uname -r)
CONFIG_HAVE_ARCH_SECCOMP_FILTER=y
CONFIG_SECCOMP_FILTER=y
CONFIG_SECCOMP=y

cat > /etc/security/limits.d/99-sonarqube.conf <<EOF
sonarqube   -   nofile   65536
sonarqube   -   nproc    65535
EOF

# sonarqube es需要安装安装jdk11
yum -y install java-11-openjdk.x86_64


# 7.9最新版本不支持mysql，数据库支持MSSQL/Oracle/PostgreSQL
# 安装PostgreSQL
# 创建sonarqube用户，授权用户create, update, and delete权限
# 如果想自定义数据库名称，不用pulic，则需要搜索路径修改
yum install -y https://download.postgresql.org/pub/repos/yum/9.6/redhat/rhel-7-x86_64/pgdg-centos96-9.6-3.noarch.rpm
yum install -y postgresql96-server postgresql96-contrib
/usr/pgsql-9.6/bin/postgresql96-setup initdb
systemctl start postgresql-9.6
systemctl enable postgresql-9.6
su - postgres
psql
create user sonarqube with password 'sonarqube';
create database sonarqube owner sonarqube;
grant all  on database sonarqube to sonarqube;
\q 

# 查看postgresql监听
vi /var/lib/pgsql/9.6/data/postgresql.conf

# 配置白名单
vi /var/lib/pgsql/9.6/data/pg_hba.conf
host    all              all             127.0.0.1/32           md5
#重启服务
systemctl restart postgresql-9.6

ss -tan | grep 5432
# 创建库/用户，并授权
psql -h 127.0.0.1 -p 5432  -U postgres


# 下载软件包
cd /opt && wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-7.9.1.zip
ln -sv sonarqube-7.9.1 sonarqube
chown sonarqube.sonarqube sonarqube/* -R

# 切换到系统sonarqube用户开始安装
su - sonarqube

# 设置数据库访问，编辑$SONARQUBE-HOME/conf/sonar.properties
sonar.jdbc.username=sonarqube
sonar.jdbc.password=sonarqube
# 注意为127.0.0.1
sonar.jdbc.url=jdbc:postgresql://127.0.0.1/sonarqube

# 配置ES存储路径，编辑SONARQUBE-HOME/conf/sonar.properties 
sonar.path.data=/var/sonarqube/data
sonar.path.temp=/var/sonarqube/temp

# 配置web server，编辑SONARQUBE-HOME/conf/sonar.properties
sonar.web.host=192.0.0.1
sonar.web.port=80
sonar.web.context=/sonarqube

# web服务器性能调优
$SONARQUBE-HOME/conf/sonar.properties
sonar.web.javaOpts=-server


$SONARQUBE-HOME/conf/wrapper.conf 
wrapper.java.command=/path/to/my/jdk/bin/java


# 执行启动脚本
Start:
$SONAR_HOME/bin/linux-x86-64/sonar.sh start

Graceful shutdown:
$SONAR_HOME/bin/linux-x86-64/sonar.sh stop

Hard stop:
$SONAR_HOME/bin/linux-x86-64/sonar.sh force-stop

# 插件安装
1.Marketplace方式安装（Administration > Marketplace）
2.手动安装（将下载好的插件上传至服务器目录：$SONARQUBE_HOME/extensions/plugins，重启sonarqube服务）
```

### 2.1.3 docker方式部署

```shell
docker pull sonarqube

docker run -d --name sonarqube -p 9000:9000 sonarqube

# 分析mvn项目
# On Linux:
$ mvn sonar:sonar

# With boot2docker:
$ mvn sonar:sonar -Dsonar.host.url=http://$(boot2docker ip):9000

# docker主机系统要求
sysctl -w vm.max_map_count=262144
sysctl -w fs.file-max=65536
ulimit -n 65536
ulimit -u 4096
```

## 2.2 Gitlab集成

### 2.2.1 sonar-scanner安装

由于gitlab项目较多，共用了gitlab-runner，因此在gitlab-runner安装sonner-scanner即可，可通用对构建的项目进行扫描

```shell
# 下载安装
cd /opt && wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.0.0.1744-linux.zip

# 添加进PATH
mv sonar-scanner-4.0.0.1744-linux sonar-scanner
cat > /etc/profile.d/sonar-scanner.sh <<EOF
export PATH=$PATH:/opt/sonar-scanner/bin
EOF
source /etc/profile.d/sonar-scanner.sh

[root@common-runner ~]# sonar-scanner -h
INFO: 
INFO: usage: sonar-scanner [options]
INFO: 
INFO: Options:
INFO:  -D,--define <arg>     Define property
INFO:  -h,--help             Display help information
INFO:  -v,--version          Display version information
INFO:  -X,--debug            Produce execution debug output
```

### 2.2.2 sonarqube web配置项目

Create new project->Provide a token->

![image-20190810103308607](/Users/xuel/Library/Application Support/typora-user-images/image-20190810103308607.png)

![image-20190810104422957](/Users/xuel/Library/Application Support/typora-user-images/image-20190810104422957.png)

### 2.2.3 配置gitlab-ci

Gitlab ci/cd发布 vue项目

```shell
stages:
  - sonarqube_scan
  - deploy_src
  - check_server

variables:
  RUNNER_HOME: "/home/gitlab-runner/builds/1sWu8AdN/0/haoxing"
  BASE_DIR: "/smartms_frontendadmin/"

job sonarqube_scan_job:
  stage: sonarqube_scan
  script:
    - sonar-scanner -Dsonar.projectKey=smartms_frontendadmin_test -Dsonar.sources=. -Dsonar.host.url=http://43.254.54.93:9000 -Dsonar.login=a393276921f1f3307a316dd88a7fe5203004a714
  tags:
    - smartms-frontend-test
  only:
    - develop
  when: always

job deploy_src_job:
  stage: deploy_src
  cache:
    paths:
    - node_modules/
  script:
    - cnpm install
    - cnpm install vue-cli
    - cnpm run build
    - rsync -az --delete ${RUNNER_HOME}${BASE_DIR}dist/ /data/dist/
  tags:
    - smartms-frontend-test
  only:
    - develop
  when: always

job check_server_job:
  stage: check_server
  script:
    - netstat -lntup |grep nginx
  tags:
    - smartms-frontend-test
  only:
    - develop
  when: always

```



发布python项目

```shell
stages:
  - sonarqube_scan
  - deploy_src
  - install_dependency
  - restart_server
  - check_server

variables:
  RUNNER_BASE_DIR: "/home/gitlab-runner/builds/QFafxxxEq/0/devops/"
  BASE_DIR: "/go2cloud_api/"

job sonarqube_scan_job:
  stage: sonarqube_scan
  # 注意，此用户为gitlab-runner执行，指定/.为此项目目录
  script:
    - sonar-scanner -Dsonar.projectKey=go2cloud_api_test -Dsonar.sources=/. -Dsonar.host.url=http://43.xxx.xxx.xxx:9110 -Dsonar.login=a393276xxxxxxxxxxxxxxxxxxx03004a714
  tags:
    - 51common-runner
  only:
    - go2cloud-platform-test
  when: always

job deploy_src_job:
  stage: deploy_src
  script:
    - scp -r ${RUNNER_BASE_DIR}${BASE_DIR}* root@172.16.100.5:/project${BASE_DIR}
  tags:
    - 51common-runner
  only:
    - go2cloud-platform-test
  when: always

```

![image-20190811171410920](/Users/xuel/Library/Application Support/typora-user-images/image-20190811171410920.png)

提交代码测试：

![image-20190810120432196](/Users/xuel/Library/Application Support/typora-user-images/image-20190810120432196.png)



查看运行job

![image-20190810120451519](/Users/xuel/Library/Application Support/typora-user-images/image-20190810120451519.png)

查看sonarqube项目

![image-20190810110741896](/Users/xuel/Library/Application Support/typora-user-images/image-20190810110741896.png

![image-20190810112146155](/Users/xuel/Library/Application Support/typora-user-images/image-20190810112146155.png)

查看详情

![image-20190810120530281](/Users/xuel/Library/Application Support/typora-user-images/image-20190810120530281.png)

## 2.3 jenkins集成

可以利用插件集成，也可以将sonar-scanner 安装在jenkins服务区上面，每次进行工具扫描。

### 2.3.1 sonar-scanner安装

sonar-scanner安装和gitlab-runner上安装一样，详见：2.2.1 sonar-scanner安装

可以两种方式集成：直接在构建的时候执行扫描命令分析报告，插件形式集成。

### 2.3.2 集成job

#### 2.3.2.1 脚本构建集成

在构建的时候利用安装好的sonar-scanner命令集成

```shell
# 配置PATH
export PATH=/data/apps/miniconda3/bin:/data/apps/miniconda3/condabin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/data/apps/miniconda3/bin:/data/apps/software/sonar-scanner/bin:/root/bin
# 指定jenkins的workspace目录
BASE_DIR=/root/.jenkins/workspace/
# 指定项目名称，此job的名称
PROJECT=go2cloud_api_prod_job
# 指定conda中项目的虚拟环境
PROJECT_ENV=go2cloud-api-prod-env
# 切换python环境
source activate ${PROJECT_ENV}
$(which python) -m pip install mock nose coverage
# 更新python环境
echo "++++++++++++++++++++++++++++++更新Python环境+++++++++++++++++++++++++++++++++++++++"

if [ -f ${BASE_DIR}${PROJECT}/requirements.txt ];then
    $(which python) -m pip install -r ${BASE_DIR}${PROJECT}/requirements.txt && echo 0 || echo 0
fi

# 代码检查/单元测试/代码测试覆盖率
echo "+++++++++++++++++++++++++++++++代码检查+++++++++++++++++++++++++++++++++++++++"
cd ${BASE_DIR}
# 生成pylint.xml
$(which pylint) -f parseable --disable=C0103,E0401,C0302 $(find ${PROJECT}/* -name *.py) >${BASE_DIR}${PROJECT}-pylint.xml || echo 0

#echo "+++++++++++++++++++++++++++++++单元测试+++++++++++++++++++++++++++++++++++++++"
# 生成nosetests.xml
#$(which nosetests) --with-xunit --all-modules --traverse-namespace --with-coverage --cover-package=go2cloud-api-deploy-prod --cover-inclusive || echo 0
#$(which nosetests) --with-xunit --all-modules --traverse-namespace --with-coverage --py3where=${PROJECT} --cover-package=${PROJECT} --cover-inclusive || echo 0
#echo "+++++++++++++++++++++++++++++++代码覆盖率+++++++++++++++++++++++++++++++++++++++"
# 生成coverage.xml
# -m coverage xml --include=${PROJECT}* || echo 0

# sonarqube 代码扫描
sonar-scanner \
  -Dsonar.projectKey=go2cloud_api_prod \
  -Dsonar.sources=${BASE_DIR}${PROJECT}/. \
  -Dsonar.host.url=http://xxx.xxx.xxx.xxx:9100 \
  -Dsonar.login=2194d90xxxxxxxxxxxxxxxxxxxxxxxxbec7f69
```

![image-20190811171848861](/Users/xuel/Library/Application Support/typora-user-images/image-20190811171848861.png)

运行项目查看

![image-20190811171958236](/Users/xuel/Library/Application Support/typora-user-images/image-20190811171958236.png)

查看sonarqube项目

![image-20190810154933049](/Users/xuel/Library/Application Support/typora-user-images/image-20190810154933049.png)

![image-20190810155012906](/Users/xuel/Library/Application Support/typora-user-images/image-20190810155012906.png)

![image-20190810160919362](/Users/xuel/Library/Application Support/typora-user-images/image-20190810160919362.png)

#### 2.3.2.2 插件形式集成

* jenkins服务器scanner配置

```shell
# 如果在构建中未指定将扫描报告发送给server端地址，需要在客户端中配置,在安装好scanner的conf目录下修改：sonar-scanner.properties 服务端的地址
sonar.host.url=http://xxx.xxx.xxx.xxx:9100
```



* jenkins服务器安装scanner

![image-20190811172435152](/Users/xuel/Library/Application Support/typora-user-images/image-20190811172435152.png)

* 工具中配置scanner

![image-20190811172311892](/Users/xuel/Library/Application Support/typora-user-images/image-20190811172311892.png)

* sonarqube server生产token

需要在sonarqube server配置jenkins的api token，用来jenkins将报告发送给sonarqube server

![image-20190811180755391](/Users/xuel/Library/Application Support/typora-user-images/image-20190811180755391.png)

* 全局工具配置中添加SonarQube servers

jenkins利用sonarqube的token创建凭据

![image-20190811182458568](/Users/xuel/Library/Application Support/typora-user-images/image-20190811182458568.png)



![image-20190811185702577](/Users/xuel/Library/Application Support/typora-user-images/image-20190811185702577.png)

* 构建项目配置

![image-20190811191138508](/Users/xuel/Library/Application Support/typora-user-images/image-20190811191138508.png)

根据扫描的程序语言填写对应的analysis properties，在此填写项目相关信息。

如果使用pipeline，可以参考声明式示例

```shell
pipeline {
    agent any
    stages {
        stage('SonarQube analysis 1') {
            steps {
                sh 'mvn clean package sonar:sonar'
            }
        }
        stage("Quality Gate 1") {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }
        stage('SonarQube analysis 2') {
            steps {
                sh 'gradle sonarqube'
            }
        }
        stage("Quality Gate 2") {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }
    }
}

```

* 查看扫描结果

在jenkins上面配置了projectname，在sonarqube上就不用配置项目

![image-20190811191520529](/Users/xuel/Library/Application Support/typora-user-images/image-20190811191520529.png)



# 三 思考

* 集成场景由于项目后端为python，在此没有编译，java项目需要安装对应mvn或其他编译工具，
* 测试环境之间为gitlab ci/cd，正式环境利用jenkins发布
* 对比jenkins脚本集成和插件方式集成，可以发现脚本方式需要在两边都配置，而且针对不同的项目，每次都需要申请token和关联项目笔记比较麻烦，插件形式一次配置好，就不用在去sonarqube上去配置，之后配置jenkins的job只需要配置简单的projectname即可，比较方便。

# 四 错误处理

* Gitlab-runner输出限制，gitlab界面显示不全`ob's log exceeded limit of 4194304 bytes.`

  ```shell
  https://gitlab.com/gitlab-org/gitlab-runner/blob/master/docs/configuration/advanced-configuration.md#the-runners-section
  ```

  

# 五 参考链接：

* [Docker镜像](https://hub.docker.com/_/sonarqube/)
* [官方文档](https://docs.sonarqube.org/latest/setup/get-started-2-minutes/)
* [postgresql常用命令](https://www.cnblogs.com/zhoujie/p/pgsql.html)

* [sonarqube升级](https://www.cnblogs.com/mascot1/p/11295048.html)