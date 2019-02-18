---
title: jenkins多分枝使用
date: 2018-11-04 12:12:28
tags:
- jenkins
---
## jenkins多分枝使用

### 1. 准备多分枝库

**确保每个分支中都要有`Jenkinsfile`文件**

<!--more-->
### 2. 使用buleocean

buleocean默认会创建`mutipileline`项目

在buleocean里面添加项目库，使用git的方式，仅仅支持git方式链接

添加完项目后就会自动扫描项目中的分支，并查找`Jenkinsfile`文件

然后，自动执行`Jenkinsfile`中的流程

### 3. 单分支流程图

访问`http://devops.li-rui.top:8002/blue/pipelines`

进入到刚刚创建的项目，点击`Branches`按钮

点击想要更改的分支进行操作

**选中单个分支**进入单个分支以后，点击右上方的`edit`按钮，就会出现流程图编辑界面

在流程图编辑界面右侧，可以以UI的方式编辑`Jenkinsfile`文件，编辑完成后

Jenkins会将`Jenkinsfile`文件的更改`push`到git库中

然后Jenkins再进行常规的构建流程，把库从git上面拉下来寻找分支中的`Jenkinsfile`文件，进行构建。

- 并行

在一个测试阶段使用多个测试单元时，使用的是并行

```java
    stage('test2') {
      parallel {
        stage('test2') {
          steps {
            echo 'it is 288'
          }
        }
        stage('test3') {
        steps {
            echo 'hello'
          }
        }
      }
    }
```

### 4. 多分支流程图

不同的分支可以对应不同的代码状态，比如dev分支为开发状态等

针对多分支的部署需要，让不同的分支中的`Jenkinsfile`要保持高度一致

针对不同的分支需要在`Jenkinsfile`中添加`when`判断进行分支选择

```java
pipeline {
	agent none
    stages {
			  //build
        stage('build') {
				when {
					branch 'dev'

				}
				agent { label 'master' }

						steps {
                echo 'this is build stage'

           }
        }

				//test
				stage('test') {
				agent { label 'master' }

						steps {
                echo 'this is test stage'

           }
        }
        //product
				stage('product') {
				when {
					branch 'pro'

				}
				agent { label 'master' }

						steps {
                echo 'this is product'

           }
        }

    }
}

```

**每个分支都会去执行该Jenkinsfile**

- 对master

执行`stage test`

- 对dev

执行`stage build` 和`stage test`

- 对pro

执行`stage test` 和`stage product`


这就做到了不同agen执行不同的分支，进而完成很好的测试
