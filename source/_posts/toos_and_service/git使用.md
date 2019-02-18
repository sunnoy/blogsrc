---
title: git使用
date: 2018-11-04 12:12:28
tags:
- git
---
## git使用

### git概念

git只关心文件整体数据的变化，而不是文件内容的变化。git会对每一个文件进行快照，一个发生版本变化时就会存储之前的快照，而没有发生变化的文件就会产生一个快照链接。
<!--more-->
#### 小型文件系统

**git更像是一个小型的文件系统**这是git与其他版本控制系统的重要区别

![git快照](https://qiniu.li-rui.top/git快照.png)

#### 本地执行

git在本地磁盘保存着所有当前项目的历史更新，在处理过程中不会联网，具有很快的执行速度。

#### 哈希值

git会对所有文件和目录进行哈希校验处理，生成一个40位的十六进制的字符串。git的工作完全依赖哈希字符串。

#### 文件状态

![git文件状态](https://qiniu.li-rui.top/git文件状态.png)

- 已提交(committed)，文件已经保存到本地数据库中
- 已修改(modified)，文件已经被修改，没有提交到本地数据库
- 以暂存(staged)，已经修改的文件放在下次提交的清单中

目录.git是保存元数据和对象数据的地方

工作目录：从.git目录中的压缩数据提取出来的文件目录，用来修改文件的地方

暂存区域：.git中的文件

#### 工作流程

在工作目录修改文件->对修改的文件快照，然后保存到暂存区->提交更新，将保存的暂存区永久存储到.git目录中

### git基础

#### 创建git仓库

本地目录

```bash
cd existing_folder
git init
git remote add origin git@gitlab.zzidc.com:lirui/wa.git
git add .
git commit -m "Initial commit"
git push -u origin master
```

从现有仓库

使用的是`git clone`命令，该命令收取的数据是仓库中每一个文件每一个版本的数据。该命令会在项目目录内创建.git目录，然后把最新版本的文件拷贝到项目目录内。也可对原来的项目目录更改名称

```bash
git clone git://github.com/schacon/grit.git mygrit
```

#### 记录更新到仓库

##### 工作目录的文件仅有两种状态：

- 已跟踪tracked 
纳入到版本控制系统中的文件，有快照，状态可能是未更新，已修改或者已放入暂存区

- 未跟踪Untracked 
所有其他文件都是未跟踪的文件

初次克隆一个仓库都是已跟踪，状态为未修改。

文件被跟踪以后的状态：
- 编辑过后为已修改
- 已修改后放到暂存区

![git文件循环](https://qiniu.li-rui.top/git文件循环.png)

##### 查看工作目录状态

```bash
git status

```
跟踪新文件至暂存区

```bash
#add后Changes to be committed
#add就是add file into staged area
#git add可以用它开始跟踪新文件，或者把已跟踪的文件放到暂存区，还能用于合并时把有冲突的文件标记为已解决状态等
git add README
```

已跟踪文件
已经跟踪文件发生修改后状态为：Changes not staged for commit

add过后的文件修改后需要重新add

##### 忽略文件

工作目录内创建文件.gitignore

示例

```bash
# 此为注释 – 将被 Git 忽略
# 忽略所有 .a 结尾的文件
*.a
# 但 lib.a 除外
!lib.a
# 仅仅忽略项目根目录下的 TODO 文件，不包括 subdir/TODO
/TODO
# 忽略 build/ 目录下的所有文件
build/
# 会忽略 doc/notes.txt 但不包括 doc/server/arch.txt
doc/*.txt
# 忽略 doc/ 目录下所有扩展名为 txt 的文件
doc/**/*.txt
```

##### 查看暂存未暂存的更新

git diff 是显示还没有暂存起来的改动
```bash
git diff
```

##### 提交更新

```bash
#-m 可以增加提交信息
#-a Git 就会自动把所有已经跟踪过的文件暂存起来一并提交，从而跳过 git add 步骤
git commit
```

##### 移除文件

```bash
rm file 
git rm file
```

#### 查看git历史

```bash
git log
```
#### 撤销操作

重新提交

```bash
git commit --amend
```

取消暂存文件

```bash
git reset HEAD benchmarks.rb
```
取消修改

```bash
git checkout -- benchmarks.rb
```

#### 远程仓库

查看远程仓库

```bash
git remote -v
git remote add pb
```
从远端仓库抓取数据

```bash
#此命令会到远程仓库中拉取所有你本地仓库中还没有的数据
git fetch name
```

### 分支

#### git中的对象

blob对象：文件被存入git后就会放入blob对象，对象名称为根据内容生成的哈希值

![blob对象](https://qiniu.li-rui.top/blob对象.png)

tree对象：tree对象是一组指针(索引)的集合，主要指向两方面的内容：其他的tree以及blob对象。

![tree](https://qiniu.li-rui.top/tree.png)

commit对象：有作者、提交者、注释、指向一个较大tree的索引(哈希值)。

![commit](https://qiniu.li-rui.top/commit.png)


总的关系

当使用 git commit 新建一个提交对象前，Git 会先计算每一个子目录（本例中就是项目根目录）的校验和，然后在 Git 仓库中将这些目录保存为树（tree）对象。之后 Git 创建的提交对象，除了包含相关提交信息以外，还包含着指向这个树对象（项目根目录）的指针，如此它就可以在将来需要的时候，重现此次快照的内容了。

![zong](https://qiniu.li-rui.top/zong.png)

经过多次修改过后，每一次提交过后都会有一个指向上一次提交的索引。


![manycommits](https://qiniu.li-rui.top/manycommits.png)

#### 分支是什么

**分支，仅仅是个指向commit对象的可变指针(索引)**

master就是一个索引，也是个分支

![branch](https://qiniu.li-rui.top/branch.png)

#### 多分支呢

commit对象上多个索引就是多分枝

添加分支

```bash
git branch testing
```

![manybranch](https://qiniu.li-rui.top/manybranch.png)

如何确定在哪个分支上工作

分支索引上再加一个特别的索引HEAD来标记当前工作的分支

切换命令

```bash
git checkout master
```

![head](https://qiniu.li-rui.top/head.png)

#### 多分支均提交以后


![commitmanybranc](https://qiniu.li-rui.top/commitmanybranc.png)

#### 分支合并之单线

合并前
![合并前](https://qiniu.li-rui.top/合并前.png)

合并命令

```bash
#切换到master分支
git checkout master
#合并到master
git merge hotfix
```

合并后
![合并后](https://qiniu.li-rui.top/合并后.png)

此时和可以删除分支hotfix

```bash
git branch -d hotfix
```

#### 分支合并之多线

合并iss53到master分支

合并命令

```bash
git checkout master
git merge iss53
```
底层实现

因为C5的直接父辈并不是C4，但是C5和C4的共同祖先是C2，所以两条分支的的末端和他们共同的祖先会经过一个三方的合并计算。

![合并祖先](https://qiniu.li-rui.top/合并祖先.png)

合并后

![祖先合并后](https://qiniu.li-rui.top/祖先合并后.png)

此时可以删掉分支iss53

```bash
git branch -d iss53
```

### 冲突

遇到冲突首先定位到文件

```bash
git status
```

然后打开文件，解决冲突，删除一个分支的内容

```bash
<<<<<<< HEAD
<div id="footer">contact : email.support@github.com</div>
=======
<div id="footer">
  please contact us at support@github.com
</div>
>>>>>>> iss53
```

整理成为下面解决

```bash
<div id="footer">
  please contact us at support@github.com
</div>
```

直到命令`git status`没有冲突。

### 1. 介绍文档

[Git远程操作详解](http://www.ruanyifeng.com/blog/2014/06/git_remote.html)

[常用 Git 命令清单](http://www.ruanyifeng.com/blog/2015/12/git-cheat-sheet.html)

### 2. git下载

[下载页面](https://git-scm.com/book/zh/v2/%E8%B5%B7%E6%AD%A5-%E5%AE%89%E8%A3%85-Git)


### 3. markdown语法

[基本语法](https://www.jianshu.com/p/q81RER)

- 工具

推荐使用`Atom`，外加markdown插件以及git插件

## git使用

### 4. ssh连接
- 生成密匙


```bash
ssh-keygen -C 添加注释
#复制id_psa.pub到gitserver
```

- 非22端口

在文件路径.SSH内，也就是ssh key的文件夹增加文件config

```bash
host gitlab.gainet.com
hostname gitlab.gainet.com
port 37165

```

- 本地配置用户

```bash
# 列出git配置
git config --list
#设定帐户全局
git config --global user.name "lirui"
git config --global user.email "lirui@zzidc.net"
#设定账户在仓库内
git config  user.name "lirui"
git config  user.email "lirui@zzidc.net"

#查看配置
git config --list
```

### 5. 建立仓库

- 从线上整一个新仓库到本地

```bash
git clone git@gitlab.zzidc.com:lirui/wa.git
cd wa
touch README.md
git add README.md
git commit -m "add README"
git push -u origin master

```
- 已有文件夹添加远程仓库


- 已有仓库再添加远端仓库

```bash
cd existing_repo
git remote rename origin old-origin
git remote add origin git@gitlab.zzidc.com:lirui/wa.git
git push -u origin --all
git push -u origin --tags
```
### 6. 远端仓库使用

```bash
#查看已经连接的远端仓库
git remote -v
#删除远端仓库
git remote remove origin
#更改远端仓库url
git remote set-url origin git@gitlab.gainet.com:PCP-GROUP/nor_service.git

#重命名主机名称
git remote rename origin old-origin
#添加另一个远端仓库
git remote add pcp git@gitlab.gainet.com:PCP-GROUP/ansible.git

#推送不同的远端仓库
git push <远程主机名> <本地分支名>:<远程分支名>
#-u为设定默认推送主机与分支
git push -u origin master

```

### 7. 不用输入密码
- 使用git，密匙添加
- 添加http方式时加入密码
```bash
git remote add origin http://lirui:xinyuedd12@gitlab.zzidc.com/lirui/wa.git
```

### 8. 提交
![enter image description here](http://www.ruanyifeng.com/blogimg/asset/2015/bg2015120901.png)

```bash
# 提交暂存区到仓库区
$ git commit -m [message]

# 提交暂存区的指定文件到仓库区
$ git commit [file1] [file2] ... -m [message]

# 提交工作区自上次commit之后的变化，直接到仓库区
$ git commit -a

# 提交时显示所有diff信息
$ git commit -v

# 使用一次新的commit，替代上一次提交
# 如果代码没有任何新变化，则用来改写上一次commit的提交信息
$ git commit --amend -m [message]

# 重做上一次commit，并包括指定文件的新变化
$ git commit --amend [file1] [file2] ...
```

### 9. 分支

```bash
# 列出所有本地分支
$ git branch

# 列出所有远程分支
$ git branch -r

# 列出所有本地分支和远程分支
$ git branch -a

# 新建一个分支，但依然停留在当前分支
$ git branch [branch-name]

# 新建一个分支，并切换到该分支
$ git checkout -b [branch]

# 新建一个分支，指向指定commit
$ git branch [branch] [commit]

# 新建一个分支，与指定的远程分支建立追踪关系
$ git branch --track [branch] [remote-branch]
  git branch --track dev remotes/origin/dev

# 切换到指定分支，并更新工作区
$ git checkout [branch-name]

# 切换到上一个分支
$ git checkout -

# 建立追踪关系，在现有分支与指定的远程分支之间
$ git branch --set-upstream-to [branch] [remote-branch]
  git branch --set-upstream-to pro remotes/origin/pro

# 合并指定分支到当前分支
$ git merge [branch]

# 选择一个commit，合并进当前分支
$ git cherry-pick [commit]

# 删除分支
$ git branch -d [branch-name]

# 删除远程分支
$ git push origin --delete [branch-name]
$ git branch -dr [remote/branch]

#推送新建分支到远程
git push origin 分支名称

#拉取远程分支，并在本地创建本地分支
git fetch origin 远程分支:本地分支

#查看本地与远程分支追踪关系
git reflog show dev

```

- 覆盖本地

```bash
#强制拉取覆盖本地
git fetch --all
git reset --hard origin/master
git pull
```

### 10. webhook

- 使用flask

```bash
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
python get-pip.py

yum -y install epel-release
yum -y install python-pip
#安装flask
pip install Flask

```

- hook代码

```python
#!flask/bin/python
# encodig=utf-8
# _*_ coding:utf-8 _*_
from flask import Flask, jsonify, request, abort
import json
import os

app = Flask(__name__)


@app.route('/', methods=['GET', 'POST'])
def call_analysis():
    if request.method == 'GET':
        return 'ok you win'
    elif request.method == 'POST':
        #data = request.data
        #body = json.loads(data)

        os.system("/usr/bin/sh /opt/webhook/pull.sh")
        #return jsonify(body)
        return "ok well done"


if __name__ == '__main__':
    app.run(host='0.0.0.0', port='8003')

```

- 脚本pull.sh

```bash
cd /root/devops-playbooks/

#git fetch --all

#git reset --hard origin/master

git pull

```

### 冲突合并

```bash
#查看冲突文件
git status
#添加冲突文件
git add files
#提交冲突文件
git commit -m "add"
#推送文件
git push
```
