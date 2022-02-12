# Jenkins详解

## 一 Jenkins概述

### 1 定义步骤

#### 1.1 声明式

```shell
pipeline {
	agent any
	stages {
		stage {
			steps {
			
			}
			steps {
			
			}
			...
		}
	}
	stage {
	
	}
	...
}
```

#### 1.2 脚本式

```shell
node {
    stage('Build') {
        echo 'Building....'
    }
    stage('Test') {
        echo 'Building....'
    }
    stage('Deploy') {
        echo 'Deploying....'
    }
}
```

### 2 执行环境

#### 2.1 声明式

```shell
pipeline {
		# 定义执行环境
    agent {
        docker { image 'node:7-alpine' }
    }
    stages {
        stage('Test') {
            steps {
                sh 'node --version'
            }
        }
    }
}
```

#### 2.2 脚本式

``` shell
node {
    /* Requires the Docker Pipeline plugin to be installed */
    docker.image('node:7-alpine').inside {
        stage('Test') {
            sh 'node --version'
        }
    }
}
```

### 3 环境变量

#### 3.1 声明式

```shell
pipeline {
    agent any
		# 使用环境变量
    environment {
        DISABLE_AUTH = 'true'
        DB_ENGINE    = 'sqlite'
        # ak信息
        AWS_ACCESS_KEY_ID     = credentials('AWS_ACCESS_KEY_ID')
    		AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
    		# 用户名和密码credentials仅适用于声明性Pipeline
    		SAUCE_ACCESS = credentials('sauce-lab-dev')
    }
    #声明式Pipeline支持开箱即用的参数，允许Pipeline在运行时通过parameters指令接受用户指定的参数。使用脚本Pipeline配置参数是通过properties步骤完成的，可以在代码段生成器中找到。
		#如果您使用“使用构建参数”选项来配置Pipeline以接受参数，那么这些参数可作为params 变量的成员访问。
		#	假设一个名为“Greeting”的String参数已经在配置中 Jenkinsfile，它可以通过${params.Greeting}以下方式访问该参数：
    parameters {
        string(name: 'Greeting', defaultValue: 'Hello', description: 'How should I greet the world?')
    }

    stages {
        stage('Build') {
            steps {
                sh 'printenv'
            }
        }
    }
}
```

#### 3.2 脚本式

```shell
properties([parameters([string(defaultValue: 'Hello', description: 'How should I greet the world?', name: 'Greeting')])])
node {
    withEnv(['DISABLE_AUTH=true',
             'DB_ENGINE=sqlite']) {
        stage('Build') {
            sh 'printenv'
        }
    }
}
```

### 4 测试清理

#### 4.1 声明式

```shell
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh './gradlew build'
            }
        }
        stage('Test') {
            steps {
                sh './gradlew check'
            }
        }
    }
    post {
        always {
            junit 'build/reports/**/*.xml'
            archive 'build/libs/**/*.jar'
            junit 'build/reports/**/*.xml'
        }
        success {
            echo 'I succeeeded!'
        }
        unstable {
            echo 'I am unstable :/'
        }
        failure {
            echo 'I failed :('
        }
        changed {
            echo 'Things were different before...'
        }
        failure {
        mail to: 'team@example.com',
             subject: "Failed Pipeline: ${currentBuild.fullDisplayName}",
             body: "Something is wrong with ${env.BUILD_URL}"
    }
    }
}
```

#### 4.2 脚本式

```groovy
node {
    try {
        stage('No-op') {
            sh 'ls'
        }
    }
}
catch (exc) {
    echo 'I failed'
}
finally {
    if (currentBuild.result == 'UNSTABLE') {
        echo 'I am unstable :/'
    } else {
        echo 'One way or another, I have finished'
    }
}
```

## 二 Pipeline

### 1 概述

#### 1.1 概念

Jenkins Pipeline是一套插件，支持将连续输送Pipeline实施和整合到Jenkins。Pipeline提供了一组可扩展的工具，用于通过PipelineDSL为代码创建简单到复杂的传送Pipeline。 

复杂的Pipeline难以在Pipeline配置页面的文本区域内进行写入和维护。为了使这更容易，Pipeline也可以写在文本编辑器中，并检查源控件，作为Jenkinsfile，Jenkins可以通过Pipeline脚本从SCM选项加载的控件

为此，在定义Pipeline时，从SCM中选择Pipeline脚本。

选择SCM选项中的Pipeline脚本后，不要在Jenkins UI中输入任何Groovy代码; 您只需指定要从其中检索Pipeline的源代码中的路径。更新指定的存储库时，只要Pipeline配置了SCM轮询触发器，就会触发一个新构建。

#### 1.2 示例

```groovy
pipeline {
		#agent 表示Jenkins应该为Pipeline的这一部分分配一个执行者和工作区。
    agent any 
    
		
    stages {
    		#stage 描述了这条Pipeline的一个阶段
        stage('Build') { 
        
        		#steps 描述了要在其中运行的步骤 stage
            steps { 
            		#sh 执行给定的shell命令
                sh 'make' 
            }
        }
        stage('Test'){
            steps {
                sh 'make check'
                #junit是由JUnit插件提供的 用于聚合测试报告的Pipeline步骤。
                junit 'reports/**/*.xml' 
            }
        }
        stage('Deploy') {
            steps {
              	# 使用内联shell conditional（sh 'make || true'）确保该 sh步骤始终看到零退出代码，从而使该junit步骤有机会捕获和处理测试报告。下面的“ 处理故障”部分将详细介绍其他方法。
                sh 'make check || true' 
        				junit '**/target/*.xml' 
            }
        }
    }
}
```

#### 1.3 功能

* 代码：Pipeline以代码的形式实现，通常被检入源代码控制，使团队能够编辑，审查和迭代其传送流程。

* 耐用：Pipeline可以在计划和计划外重新启动Jenkins管理时同时存在。

* Pausable：Pipeline可以选择停止并等待人工输入或批准，然后再继续Pipeline运行。

* 多功能：Pipeline支持复杂的现实世界连续交付要求，包括并行分叉/连接，循环和执行工作的能力。

* 可扩展：Pipeline插件支持其DSL的自定义扩展 以及与其他插件集成的多个选项。

#### 1.4 语法学习

* 内置语法

  *  [localhost：8080 / pipeline-syntax /](http://localhost:8080/)

* 代码生成器

  * 内置的“Snippet Generator”实用程序有助于为单个步骤创建一些代码，发现插件提供的新步骤，或为特定步骤尝试不同的参数。

    Snippet Generator动态填充Jenkins实例可用的步骤列表。可用的步骤数量取决于安装的插件，它明确地暴露了在Pipeline中使用的步骤。

### 2 语法





### 3 BlueOcean



#### 3.1 概念

BlueOcean重新考虑了Jenkins的用户体验。BlueOcean由Jenkins Pipeline设计，但仍然兼容自由式工作，减少了团队成员的混乱，增加了清晰度。

- 连续交付（CD）Pipeline的复杂可视化，允许快速和直观地了解Pipeline的状态。
- Pipeline编辑器通过引导用户直观和可视化的过程创建Pipeline，使创建Pipeline平易近人。
- 个性化，以适应团队每个成员的角色需求。
- 需要干预和/或出现问题时确定精度。BlueOcean显示了Pipeline需要注意的地方，便于异常处理和提高生产率。
- 用于分支和拉取请求的本地集成可以在GitHub和Bitbucket中与其他人进行代码协作时最大限度提高开发人员的生产力。



#### 3.2 安装

BlueOcean可以安装在现有的Jenkins环境中，也可以使用[Docker](https://www.w3cschool.cn/docker/)进行运行 。

要开始在现有的Jenkins环境中使用Blue Ocean插件，它必须运行Jenkins 2.7.x或更高版本：

1. 登录到您的Jenkins服务器
2. 单击边栏中的管理Jenkins然后管理插件
3. 选择可用的选项卡，并使用搜索栏查找BlueOcean
4. 单击“安装”列中的复选框
5. 单击安装而不重新启动或立即下载并重新启动后安装

![image-20190630121302890](/Users/xuel/Library/Application Support/typora-user-images/image-20190630121302890.png)













### 参考链接:

* [Grooy语法](https://www.w3cschool.cn/groovy/groovy_basic_syntax.html)

* [jenkins笔记](https://www.w3cschool.cn/jenkins/jenkins-173a28n4.html)

  

  