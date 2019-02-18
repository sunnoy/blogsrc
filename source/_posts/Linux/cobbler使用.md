---
title: cobbler使用
date: 2018-11-04 12:12:28
tags:
- cobbler
---
## cobbler使用

### 架构

![fig-1](https://qiniu.li-rui.top/fig-1.png)

## 硬件要求

客户机和cobbler机器内存要在**2GB**以上
<!--more-->
### 安装

```bash
rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
yum -y install cobbler cobbler-web dhcp tftp-server pykickstart httpd xinetd
```

## 安全相关

```bash
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
setenforce 0
systemctl stop firewalld 
systemctl disable firewalld
```

### 配置

#### 文件 /etc/cobbler/settings

```yml
#cobbler服务器的IP
server: 10.0.0.7

#dhcp服务器next_server
next_server: 10.0.0.7

#激活管理dhcp
manage_dhcp: 1

#服务器pxe为第一启动的时候，可以防止循环装系统
pxe_just_once: 1

#
```

#### 下载一些loader

```bash
#loadr会下载到/var/lib/cobbler/loaders/
cobbler get-loaders 
```

#### 

```bash
vi /etc/xinetd.d/tftp
disable = no

service xinetd restart
```

#### 常用命令

```bash
cobbler check    #核对当前设置是否有问题
cobbler list     #列出所有的cobbler元素
cobbler report   #列出元素的详细信息
cobbler sync     #同步配置到数据目录,更改配置最好都要执行下
cobbler reposync #同步yum仓库
cobbler distro   #查看导入的发行版系统信息
cobbler system   #查看添加的系统信息
cobbler profile  #查看配置信息
```
#### 重新装系统

删掉相应的system，然后重新添加system，这样是为了清除DHCP记录。然后sync，然后客户机重启就行了

## 镜像

### 导入镜像

```bash
cobbler import --path=/run/media/root/CentOS\ 7\ x86_64/ --name=CentOS-7.4-1708-x86_64 --arch=x86_64  

#--path路径
#--name为发行版系统的唯一标识
#--arch支持x86│x86_64│ia64

#该镜像在位置/var/www/cobbler/ks_mirror
```

导入镜像后 

查看镜像列表

```bash
cobbler distro list
```

### 生成

#### 生成工具

```bash
yum install -y system-config-kickstart
```

#### 手动编辑

指定ks文件

```bash
cobbler profile edit --name=CentOS-7.4-1708-x86_64 --kickstart=/var/lib/cobbler/kickstarts/CentOS-7.4-1708-x86_64.ks
```

#### 查看

```bash
cobbler profile report CentOS-7.4-1708-x86_64
```

## 服务检查

```bash
systemctl restart xinetd
systemctl restart httpd
systemctl restart cobblerd
systemctl restart dhcpd

```

cobbler是由httpd来负责代理的，安装走的httpd协议，dhcp负责为机器分配IP，xinetd则是用来管理tfpt服务，tftp是一种小型的文件传输服务。

cobbler里面设定的IP一定要在cobbler的服务器上配置，不然无法启动

## 定制化主机安装

system用来进行定制化安装，需要指定一个ks，一个profile，然后加上客户机的mac来安装，

### 导入iso

```bash
mount -t iso9660 -o loop CentOS-7-x86_64-Minimal-1804.iso /mnt/
cobbler import --path=/mnt --name=centos1804 --arch=x86_64
```

### 创建定制的profile

主要是绑定ks文件

```bash
cobbler profile add --name=Fedora17-xfce \
                    --ksmeta='desktop_pkg_group=@xfce-desktop' \
                    --kickstart=/var/lib/cobbler/kickstarts/example.ks \
                    --parent=Fedora17-x86_64
```

### 添加system

用来定制化安装，对不同的主机安装分配不同的mac地址

```bash
cobbler system add --name=matr01 --mac=00:0C:29:F0:87:FE --profile=CentOS-7.4-1708-x86_64 \
--ksmeta="net_ip=207"
--ip-address=192.168.178.50 --subnet=255.255.255.0 --gateway=192.168.178.2 --interface=ens33 \
--static=1 --hostname=matr01 --name-servers="114.114.114.114" \
--kickstart=/var/lib/cobbler/kickstarts/m01.ks

#ksmeta用于在ks文件中传递变量
#使用如下
echo $net_ip

```

### 最后同步一下

```bash
#同步
cobbler sync

#查看
cobbler list
```

## repo 

### 添加

```bash
cobbler repo add --name=centos7-base --mirror=http://mirrors.163.com/centos/7/os/x86_64/ --arch=x86_64 --breed=yum --mirror-locally=N

cobbler repo add --name=centos7-epel --mirror=https://mirrors.aliyun.com/epel/7Server/x86_64/ --arch=x86_64 --breed=yum --mirror-locally=N
```

### 同步源

这个是把远端的源同步到本地，然后形成本地源，然后cobbler会配置相关repo

使用reposync同步，同步的文件在目录/var/www/cobbler/repo_mirror内，**注意磁盘空间**

```bash
cobbler reposync
```

### 加入profile

```bash
cobbler profile add --name=centos7-repo --repos="centos7-base centos7-epel"  --distro=CentOS-7.4-1708-x86_64  
		--kickstart=/var/lib/cobbler/kickstarts/m01.ks
```

### 配置生效

```bash
#vim /etc/cobbler/settings
yum_post_install_mirror: 1

#ks文件中存在
$yum_config_stanza

```

### 效果

系统安装完成后会在目录/etc/yum.repos.d/内创建文件cobbler-config.repo ，内容如下

```bash
#发行版的自带源
[core-0]
name=core-0
baseurl=http://172.17.10.14/cobbler/ks_mirror/centos5.8-x86_64
enabled=1
gpgcheck=0
priority=1
#添加的源
[centos5.8-x86_64-base]
name=centos5.8-x86_64-base
baseurl=http://172.17.10.14/cobbler/repo_mirror/centos5.8-x86_64-base
enabled=1
priority=99
gpgcheck=0
```

## web

### 相关依赖

需要用到Django1.8.9

```bash
wget https://bootstrap.pypa.io/get-pip.py
python get-pip.py
pip install Django==1.8.9
systemctl restart httpd
```

直接web访问：https://cobbler_ip/cobbler_web

用户名称密码 都是cobbler

## ansible

### 准备

动态指定脚本

```bash
#动态的hosts，
ansible -i cobbler.py all -m ping
https://raw.github.com/ansible/ansible/devel/contrib/inventory/cobbler.py

```

连接cobbler api配置

```ini
#cobbler.ini和cobbler.py在同级目录内
[cobbler]

# Set Cobbler's hostname or IP address
host = http://127.0.0.1/cobbler_api

# API calls to Cobbler can be slow. For this reason, we cache the results of an API
# call. Set this to the path you want cache files to be written to. Two files
# will be written to this directory:
#   - ansible-cobbler.cache
#   - ansible-cobbler.index

cache_path = /tmp

# The number of seconds a cache file is considered valid. After this many
# seconds, a new API call will be made, and the cache file will be updated.

cache_max_age = 900
```

### cobbler

对profile添加kemeta

```bash
cobbler profile add --name=webserver --distro=CentOS-7.4-1708-x86_64
cobbler profile edit --name=webserver --mgmt-classes="webserver" --ksmeta="a=2 b=3"
```

## snippets

**sinippets中不可以随便使用带变量的脚本，会有冲突导致装机失败**

### 添加key

```bash
#/var/lib/cobbler/snippets/add_host_ssh_keys
cd /root
mkdir --mode=700 .ssh
cat >> .ssh/authorized_keys << "PUBLIC_KEY"
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCYlc2znSI1QLecRDaX1sjqpH9KX3Uwg29kWpODv1rnqH+4ljPXGCmWS10H9BEQ/CGge8pNYKvQ2VfyOoYUbYlRPKQykayDtBud32DaT7IIdy4g4qNPdjukSkDPlc4sIODHWCLxM1X+CBkdq8fU9GNPRoVRc9U8CqchqCLaKozunZu1hDUonduEQnDZCc9axgyrnE8rk5W4eO/aQfRaLbENxnDJ4wfRa2q4Rf15a3u9YGgZ6IeLnI+HSgBXgjM6wVhB5SSaJY+taf+8PWyd8Pb9ZZ+ZjQgXa8gGvVziMqbJ4ycDOlCX4uyY6YRnC5wiCnP8q3huYUKAWAU/ypFtxFvp root@deploy-center
PUBLIC_KEY
chmod 600 .ssh/authorized_keys

sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/g' /etc/ssh/sshd_config
sed -i 's/GSSAPIAuthentication yes/GSSAPIAuthentication no/g' /etc/ssh/sshd_config
sed -i 's/#UseDNS yes/UseDNS no/g' /etc/ssh/sshd_config
```

### 优化内核参数

```bash
#/var/lib/cobbler/snippets/adjustment_kernel

echo "ulimit -n 131072"          >> /etc/profile && ulimit -n 131072
echo "*                soft    nofile         131072
*                hard    nofile         131072
*                soft    nproc         131072
*                hard    nproc         131072" >> /etc/security/limits.conf
echo 'vm.swappiness = 1
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv4.neigh.default.gc_thresh1 = 1024
net.ipv4.neigh.default.gc_thresh2 = 4096
net.ipv4.neigh.default.gc_thresh3 = 8192' >> /etc/sysctl.conf
sysctl -p
echo 'NETWORKING_IPV6=no' >> /etc/sysconfig/network
sed -i 's/IPV6INIT=yes/IPV6INIT=no/g' /etc/sysconfig/network-scripts/ifcfg-e*

```














