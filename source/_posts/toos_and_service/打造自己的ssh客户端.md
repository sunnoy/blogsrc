---
title: 打造自己的ssh客户端
date: 2019-04-08 12:12:28
tags:
- linux
---

# 达到的效果

无需使用第三方ssh客户端，仅仅使用原生的mac上的终端工具，通过自定义主机名称来连接远程主机，主机名要tab自动完成

<!--more-->

# 创建配置文件

```bash
cd ~
vim .ssh/config

#这个可自定义
Host tx.k8s.config
    # 写远程主机的IP或者域名
    HostName 1.1.1.1
    #登录远程主机的用户名
    User root
    #登录远程主机响应用户的私钥
    IdentityFile ~/.ssh/id_rsa
    #端口默认 22
    Port 22
```

# 配置自动完成

```bash
cat > ~/.profile << EOF
complete -W "$(echo $(grep '^Host ' .ssh/config  | sort -u | sed 's/^ssh //'))" ssh
complete -W "$(echo $(grep '^Host ' .ssh/config  | sort -u | sed 's/^ssh //'))" scp
complete -W "$(echo $(grep '^Host ' .ssh/config  | sort -u | sed 's/^ssh //'))" ssh-copy-id
EOF
```

配置完成后

```bash
source .profile
```

# 使用

```bash
#通过# ssh进行命令补全
ssh 
```



