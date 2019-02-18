---
title: ansible安装配置
date: 2018-11-28 12:12:28
tags:
- ansible
---

# ansible安装配置

## 基本介绍

![ansible](https://ws2.sinaimg.cn/mw690/7569b997jw1fbuzu19fowj20k10c1t9n.jpg)

<!--more-->

主要组件介绍

### inventory

以YAML文本文件形式存在，定义了机器的基本信息

### modules

这个就是具体干活的家伙，Ansible把各种最基本的自动化动作『task』以Python module的形式实现，拿来就可以用了，是Ansible的最基本动作单元，例如上面webservers例子中的yum，template, service就是Ansible提供的任务模块。Ansible提供的任务模块超过400个

### roles

由Ansible和community分享提供，或自己编写。一个『role』就是一组task，相当于一个迷你playbook，这样相同的role，在不同的地方就不需要重复写task

### playload

playbook也是yaml文本文件，相当于演戏的脚本，把角色（role），任务（task），inventory串起来

## 安装

### yum安装

```bash
yum install epel-release -y
#安装一些需要组件包sshpass和python-netaddr
yum install ansible sshpass python-netaddr -y
```

### pip安装

#### pip安装

```bash
#yum安装pip
yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
yum install -y python-devel openssl-devel libffi-devel python-pip
#python 包安装
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
python get-pip.py
#升级pip
pip install --upgrade pip
```

#### ansible安装

最新版安装

```bash
pip install ansible
```

指定版本安装

```bash
#通过指定不存在版本后会列出版本号
pip install ansible==55555
#让后选择版本号就可以安装
pip install ansible==2.5.3
```

## 配置

推荐在一个playbook内使用单独的配置文件`ansible.cfg`

```bash
[defaults]
ansible_managed = Please do not change this file directly since it is managed by Ansible and will be overwritten
action_plugins = plugins/actions
callback_plugins = plugins/callback
roles_path = ./roles
log_path = /var/log/ansible.log
forks = 25
host_key_checking = False
gathering = smart
fact_caching = jsonfile
fact_caching_connection = $HOME/ansible/facts
fact_caching_timeout = 600
nocows = 1
callback_whitelist = profile_tasks
retry_files_enabled = False
timeout = 60
[ssh_connection]
control_path = %(directory)s/%%h-%%r-%%p
ssh_args = -o ControlMaster=auto -o ControlPersist=600s
pipelining = True 
retries = 10
```

## 使用

ansible使用方式有两种，命令行方式和playbook方式

### 命令行

在不指定任何模块的情况下默认为`command`来执行命令

```bash
#格式
ansible testservers -m 模块 -a '参数'
#全部
ansible testservers -m command -a 'uname -n'
#简写
ansible testservers -a 'uname -n'
```

指定模块使用

```bash
ansible atlanta -m copy -a "src=/etc/hosts dest=/tmp/hosts"
```

### playbook

编写好playbook后运行

```bash
ansible-playbook -i hosts.ini site.yml
```

### playbook语法检查

```bash
ansible-playbook --syntax-check site.yml
```

