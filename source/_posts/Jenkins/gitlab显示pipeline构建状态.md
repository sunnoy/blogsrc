---
title: gitlab显示pipeline构建状态
date: 2019-05-22 12:12:28
tags:
- jenkins
---

让Jenkins中pipeline构建状态在gitlab中显示

其实是Jenkins通过token的方式对接gitlab api来传递构建状态信息

<!--more-->

# token配置

## gitlab 获取token

在用户设置里面添加token

![token](https://qiniu.li-rui.top/token.png)

## Jenkins添加token

### 首先安装gitlab插件 

### 配置token

Manage Jenkins - system config - gitlab - 点击 add - 添加token

![add token](https://qiniu.li-rui.top/add%20token.png)

配置完成后注意测试是否为成功连接

![test](https://qiniu.li-rui.top/test.png)


# gitlab项目

在gitlab项目根目录准备Jenkinsfile

```Jenkinsfile
pipeline {
    agent any
    post {
      failure {
        echo "failure"
      }
      success {
        echo "success"
      }
    }
    options {
      gitLabConnection('xygit')
      gitlabBuilds(builds: ['build', 'make'])
    }
    triggers {
        gitlab(
          triggerOnPush: true, 
          secretToken: "abc"
          )
    }
    stages {
      stage("build") {
        steps {
          gitlabCommitStatus('build') {
             echo "build stage www"
          }
        }

      }

      stage("make") {
        steps {
          gitlabCommitStatus('make') {
             echo "make stage where is name"
          }
        }
      }
    }
  
}
```

# 添加项目

## blue ocean

在Jenkins中安装blue ocean插件

然后进入blue ocean 添加 pipeline

![add](https://qiniu.li-rui.top/add.png)


## 配置web hook

### 获取webhook地址
在Jenkins中找到项目地址，然后进入构建分支，查看配置

![viwe](https://qiniu.li-rui.top/viwe.png)

找到触发地址，打码的位置

![webhookurl](https://qiniu.li-rui.top/webhookurl.png)

### gitlab配置地址

进入项目页面

![addhokurl](https://qiniu.li-rui.top/addhokurl.png)

添加地址，以及选择相应的触发事件，并保存

![gitlabwebhook](https://qiniu.li-rui.top/gitlabwebhook.png)

# 测试

在gitlab进行push操作提交页面会显示

![oktest](https://qiniu.li-rui.top/oktest.png)





