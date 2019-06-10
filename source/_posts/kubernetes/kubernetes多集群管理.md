---
title: kubernetes多集群管理
date: 2019-6-10 12:12:28
tags:
- kubernetes
---

使用多个脚本工具来轻松管理多个集群，以及轻松切换集群以及命名空间。

<!--more-->

# 所需工具

- mac
- zsh mac自带
- [brew](https://brew.sh/)
- [iterm2](https://www.iterm2.com/) 
- [oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh)
- [kubectx](https://github.com/ahmetb/kubectx)
- [kube-ps1](https://github.com/robbyrussell/oh-my-zsh/tree/master/plugins/kube-ps1)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-macos)
- [kubectl plugin](https://github.com/robbyrussell/oh-my-zsh/tree/master/plugins/kubectl)
- kubectl config 文件

# 安装iterm2 

到站点上直接下载安装即可，可以参考另一篇的关于[iterm2的配置](https://www.li-rui.top/2019/06/07/toos_and_service/iterm2%E4%BE%9D%E6%8D%AEssh%20config%E5%88%9B%E5%BB%BA%E5%8A%A8%E6%80%81profile/)

# 安装oh-my-zsh

直接安装，会提示使用zsh为默认的shell，输入y既可。

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

# 安装brew

brew是mac上的包管理器

```bash
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

# 配置kubectl

## 下载kubectl

```bash
brew install kubernetes-cli
```

## 配置kubectl插件

主要是进行oh-my-zsh配置插件，可以使用tab提示出资源名称

```bash
vim .zshrc

plugins=(... kubectl)
```

## 配置kube-ps1插件

这个插件在终端左侧显示当前的集群以及当前所在的namespace

```bash
vim .zshrc

plugins=(... kubectl kube-ps1)
```

**重要**

需要将下面的字符放到，.zshrc 的末尾，这个是为了不让变量值覆盖

```bash
vim .zshrc

# 放到文件尾部 很重要
PROMPT=$PROMPT'$(kube_ps1)'
```

## 配置kubectx插件

该插件用来快速切换集群和namespace

插件在kubectl配置完成以后也是可以进行自动完成的

```bash
brew install kubectx
```

# 集群切换

## 准备config文件目录

```bash
mkdir ~/.kube
```

## 添加远程集群配置

复制远程集群上的 .kube/config 文件，需要修改几个地方

需要明白的是文件的 context包含 cluster name namespace

将修改好的文件放到mac的目录 ~/.kube 下面，假设配置了两个集群 a b

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: 
    xxx
    server: https://10.x.x.xxx:6443
  # 这个需要修改
  name: cluster-name-a
contexts:
- context:
    #context中cluster和user要保持一致
    cluster: cluster-name-a
    user: user-name-a
  #context name不要和其他的重复了  
  name: context-name-a
#这条删除即可
#current-context: context1
kind: Config
preferences: {}
users:
- name: user-name-a
  user:
    client-certificate-data: 
    xxx
    client-key-data: 
    xxx
```

## KUBECONFIG变量配置

kubectl会总本机的KUBECONFIG变量读取config信息，因此需要将多个集群的配置文件放到这个变量里面去，这里直接放到 .zshrc 里面

```bash
vim .zshrc

#多个集群注意使用 : 隔开
export KUBECONFIG=$KUBECONFIG:~/.kube/a:~/.kube/b
```

# 切换多集群

## kube-ps1 开启

使用命令 kubeon 来开启显示，效果如下 ,使用 kubeoff关闭

使用 kubectx 来切换集群，使用 kubens 来切换namespace

```bash
➜  ~ (⎈ a:default)
```

## 我的个性化配置

```bash
vim .zshrc

KUBE_PS1_SYMBOL_USE_IMG=true
KUBE_PS1_SEPERATOR=-
KUBE_PS1_PREFIX=[
KUBE_PS1_SUFFIX=]
```

# 清零

```bash
uninstall_oh_my_zsh

brew uninstall kubernetes-cli

brew uninstall kubectx

brew cleanup

ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/uninstall)"
```

# 其他插件

## 自动补全

```bash
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions

plugins=(zsh-autosuggestions)
```

## 语法高亮

```bash
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting

plugins=(zsh-syntax-highlighting)
```

