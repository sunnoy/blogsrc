---
title: jenkins pipeline使用
date: 2018-11-04 12:12:28
tags:
- jenkins
---
## jenkins pipeline使用

### 1. 基本介绍

- 含义

就是一套运行于Jenkins上的工作流框架，将原本独立运行于单个或者多个节点的任务连接起来，实现单个任务难以完成的复杂发布流程。

Pipeline的实现方式是一套Groovy DSL，任何发布流程都可以表述为一段Groovy脚本，并且Jenkins支持从代码库直接读取脚本，从而实现了Pipeline as Code的理念。

<!--more-->

- 基本概念

**stage**一个pipeline可以划分多个stage，每个stage代表一组操作，是一个逻辑分组概念

**node**为Jenkins节点，类型为master/agent，执行step的运行环境

**step**基本操作单元，有Jenkins插件提供

### 2.  编写语法

**Declarative Pipeline** Jenkins 2.5以后加入的结构化方式
适用于bule ocean，推荐使用

**Scripted Pipeline** Jenkins 2.5 之前的编写方法

#### 2.1 Declarative语法

- pipeline块

所有的指令代码必须在`pipeline`块内

```java

pipeline{

  /* insert Declarative Pipeline here */
}

```

#### 2.2 sections

sections包含一个或者多个`Directives`与`Steps`

- agent

Jenkins分布式部署中的其他有Jenkins环境的节点，多节点需要添加master意外的节点，单节点的话直接any

必须要的参数

示例

```java

//顶层指定
pipeline {
    agent { any }
    stages {
        stage('Example Build') {
            steps {
                sh 'mvn -B clean verify'
            }
        }
    }
}

//stages中指定
pipeline {
    //此处指定none，然后在stage中分别指定agent
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

- post

stages/steps执行完成后的需要执行的模块，该模块会根据前面的stages/steps执行情况与自身的条件参数来执行

```java
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
    post {
       //always/changed/aborted/successcleanup等可以选择
        always {
            echo 'I will always say Hello again!'
        }
    }
}

```

- stages

包含一个或者多个stage，多个stage的封装

```java
pipeline {
    agent any
    //stages和agent是平级缩进的
    stages {
       //至少要有一个stage
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
}
```

- steps

在steps中使用Jenkins模块来处理部署/测试/交付流程

模块中的指令参考
```bash
https://jenkins.io/doc/pipeline/steps/

```
#### 2.3 Directives

pipeline中的其他指令

- environment

在steps执行的过程中对整个pipeline中的变量添加

```java
pipeline {
    agent any
    environment {
        CC = 'clang'
    }
    stages {
        stage('Example') {
            environment {
                AN_ACCESS_KEY = credentials('my-prefined-secret-text')
            }
            steps {
                sh 'printenv'
            }
        }
    }
}
```

- options

在steps执行过程中的辅助选项比如任务超时，重试次数等

```java

pipeline {
    agent any
    options {
        //整个pipeline在1个小时内完成，
        timeout(time: 1, unit: 'HOURS')
    }
    stages {
        stage('Example') {
            options {
                timeout(time: 1, unit: 'HOURS')
            }
            steps {
                echo 'Hello World'
            }
        }
    }
}
```

- parameters

pipeline中变量

```java

pipeline {
    agent any
    parameters {
        string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')
    }
    stages {
        stage('Example') {
            steps {
                echo "Hello ${params.PERSON}"
            }
        }
    }
}

```

- triggers

pipeline中的自动化执行，`cro`, `pollSCM` 和 `upstream`。

```java

pipeline {
    agent any
    triggers {
        //定时任务
        cron('H */4 * * 1-5')

        //轮询版本控制
        //pollSCM('H */4 * * 1-5')

        //条件匹配时候触发
        upstream(upstreamProjects: 'job1,job2', threshold: hudson.model.Result.SUCCESS)

    }
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
}
```

- stage

stage在stages块内，stage包含多个steps块

```java
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
}
```

- tools

自动安装工具，并加入PATH环境变量，支持工具有`maven`,`jdk`和`gradle`

tool的名字需要在Manage Jenkins → Global Tool Configuration中配置

```java

pipeline {
    agent any
    tools {

        maven 'apache-maven-3.0.1'
    }
    stages {
        stage('Example') {
            steps {
                sh 'mvn --version'
            }
        }
    }
}
```

- input

pipeline中的提示信息

```java

pipeline {
    agent any
    stages {
        stage('Example') {
            input {
               //提示信息
                message "Should we continue?"
                //ok按钮
                ok "Yes, we should."
                //逗号分离的用户列表
                submitter "alice,bob"
                parameters {
                    string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')
                }
            }
            steps {
                echo "Hello, ${PERSON}, nice to meet you."
            }
        }
    }
}

```

- when

在stage内，when根据配置的条件来决定该stage是否需要执行

```java
...
stage('Example Deploy') {
            when {
               //判断表达式是否为true
                expression { BRANCH_NAME ==~ /(production|staging)/ }
                //anyof块中任意一个为true
                anyOf {
                    environment name: 'DEPLOY_TO', value: 'production'
                    environment name: 'DEPLOY_TO', value: 'staging'
                }
            }
...
```

- steps

steps中可以插入Scripted Pipeline

```java

pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'

                script {
                    def browsers = ['chrome', 'firefox']
                    for (int i = 0; i < browsers.size(); ++i) {
                        echo "Testing the ${browsers[i]} browser"
                    }
                }
            }
        }
    }
}
```

### 2.4 并行

```java

stage('Parallel Stage') {
            when {
                branch 'master'
            }

            //并行任务中其中一个挂掉就停止
            failFast true

            parallel {
                stage('Branch A') {
                    agent {
                        label "for-branch-a"
                    }
                    steps {
                        echo "On Branch A"
                    }
                }
                stage('Branch B') {
                    agent {
                        label "for-branch-b"
                    }
                    steps {
                        echo "On Branch B"
                    }
                }
            }
        }

```

# 构建历史自动清除

```bash
pipeline {
  options {
    buildDiscarder(logRotator(numToKeepStr: '30', artifactNumToKeepStr: '30'))
  }
  ...
}
```
