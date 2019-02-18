---
title: docker底层四件套
date: 2018-11-04 12:12:28
tags:
- docker
---
# docker底层四件套

docker在Linux中主要依赖下面四个

- Namespace
- Cgroup
- OverlayFS
- 虚拟网卡

<!--more-->
## Namespace

命名空间是系统对一系列资源的逻辑划分，也是一个特定标识符的可见范围。

docker需要重点关注的namespace：

- PID Namespace 不同Namespace中的进程编址空间彼此重合。值得注意的是，PID Namespace是一个层级结构，Child对Parent可见，反之不可见

- Net Namespace Net Namespace隔离了整个网络协议栈，包括路由表，网卡IP地址，端口号，Netfilter/iptables等在每一个Net Namespace中均是独立的

- Mount Namespace 每一个Mount Namespace均有一个独立的文件系统层级视图。

### PID Namespace

首先安装进程树命令

```bash
yum install psmisc -y
#相关选项
-a：显示每个程序的完整指令，包含路径，参数或是常驻服务的标示；
-c：不使用精简标示法；
-G：使用VT100终端机的列绘图字符；
-h：列出树状图时，特别标明现在执行的程序；
-H<程序识别码>：此参数的效果和指定"-h"参数类似，但特别标明指定的程序；
-l：采用长列格式显示树状图；
-n：用程序识别码排序。预设是以程序名称来排序；
-p：显示程序识别码；
-u：显示用户名称；
-U：使用UTF-8列绘图字符；
-V：显示版本信息。
```
我们开启一个容器然后进入容器

```bash
#启动容器
docker run -itd --rm --name test docker.li-rui.top/library/centos:7.5.1804
#执行一个bash命令，进入容器
docker exec -it test bash
```

我们查看entos的Dcokerfile，初始进程运行了bash命令

```Dockerfile
FROM scratch
ADD centos-7-docker.tar.xz /
CMD ["/bin/bash"]
```

进入容器查看里面应该有两个bash进程

```bash
#容器内
#[root@cfcfa2db0d74 /]# top
 PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
    1 root      20   0   11824   1676   1408 S   0.0  0.0   0:00.06 bash
   40 root      20   0   11824   1880   1504 S   0.0  0.0   0:00.04 bash
   60 root      20   0   56160   1968   1440 R   0.0  0.0   0:00.00 top
```

然后查看第一个bash进程的pid namespace为`pid:[4026532193]`

```bash
[root@cfcfa2db0d74 /]# ls -l /proc/1/ns/pid
lrwxrwxrwx 1 root root 0 Nov  1 02:13 /proc/1/ns/pid -> pid:[4026532193]
```

到宿主机这边，查看docker服务进程派生

```bash
[root@docker-pcp ~] pstree -p
systemd(1)─┬─agetty(1595)
           ├─auditd(7456)───{auditd}(7457)
           ├─crond(7517)
           ├─dbus-daemon(701)
           ├─dockerd(27414)─┬─docker-containe(27424)─┬─docker-containe(11561)─┬─bash(11579)
           │                │                        │                        ├─bash(12815)

```

分别查看docker派生出第一个bash的pid namespace

```bash
[root@docker-pcp ~] ls -l /proc/11579/ns/pid
lrwxrwxrwx 1 root root 0 Nov  1 10:04 /proc/11579/ns/pid -> pid:[4026532193]

```
容器内进程为1的pid namespace为4026532193。在宿主机上的pid namespace里面显示进程pid为11579，可见pid namespace和进程关系一样中存在继承关系，子PID Namespace对父PID Namespace是可见的。并且父PID Namespace可以对子PID Namespace进行操作。

![pid namespace](https://qiniu.li-rui.top/pid%20namespace.png)

### Net Namespace

net namespace就是完全隔离的,容器可以和宿主机联通是因为使用了桥接网络

来看容器中的，为`4026532199`

```bash
[root@cfcfa2db0d74 /]# ls -l /proc/1/ns/net
lrwxrwxrwx 1 root root 0 Nov  1 02:13 /proc/1/ns/net -> net:[4026532199]
```

宿主机上为`4026531956`

```bash
[root@docker-pcp ~]# ls -l /proc/1/ns/net
lrwxrwxrwx 1 root root 0 Oct 31 15:17 /proc/1/ns/net -> net:[4026531956]
```
![net namesp](https://qiniu.li-rui.top/net%20namesp.png)

### Mount Namespace 

## Linux Cgroup

通过Cgroup可对进程或者应用程序进行资源分配（如 CPU 时间、系统内存、网络带宽或者这些资源的组合）

在centos 7 中Cgroup和systemd进行了整合，整合过后会把资源管理设置从进程级别移至应用程序级别。控制更为灵活。

### systemd

systemd是大多数Linux发行版的守护进程管理工具。他就是Linux启动的1号进程，取代了cenos6中的initd。

#### 架构

systemd架构

![systemd](https://qiniu.li-rui.top/systemd.png)

一个rpm包安装完成后就会在目录`/usr/lib/systemd/system/`写入启动文件，开机启动的服务在历经`/etc/systemd/system/`内

默认情况下，systemd 会自动创建 slice、scope 和 service 单位的层级，来为 cgroup 树提供统一结构

- service 一个或一组进程，由配置文件来创建
- scope 用户会话、 容器和虚拟机被认为是scope
- slice 一组按层级排列的单位。slice 并不包含进程，但会组建一个层级，并将 scope 和 service 都放置其中。

系统默认创建的slice主要有

- -.slice —— 根 slice；
- system.slice —— 所有系统 service 的默认位置；
- user.slice —— 所有用户会话的默认位置；
- machine.slice —— 所有虚拟机和 Linux 容器的默认位置。

#### docker slice

docker没有使用系统创建的默认machine.slice，而是docker进程创建了docker slice。可见docker中的层级显示以及容器内的进程

```bash
[root@localhost ~]systemd-cgls
├─1 /usr/lib/systemd/systemd --switched-root --system --deserialize 24
├─docker
│ ├─ad088dd5fe04c441d6dd7f5e262ac37fe3249d98a48dd52b3aac254a05947033
│ │ └─3146 /home/runtime/jdk1.8.0_112/bin/java -Djava.util.logging.config.file=/home/server/apache-tomcat-7.0.6
│ ├─e0ee6e159fb148e6f153645076677e2049d7d39583caa8a9a44e66674d6c0e32
│ │ └─3104 mysqld --default-authentication-plugin=mysql_native_password
│ ├─b059bc140a277bcc21f96945007e516f3711c4468818bb42035d2d17dd5c1add
│ │ └─3092 /home/runtime/jdk1.8.0_112/bin/java -Djava.util.logging.config.file=/home/server/apache-tomcat-7.0.6
│ ├─5a36fc1e9e9ad42f1536ceecd6800a3bc5775380841f9ce5e9c00ddcb8602de2
│ │ └─2966 /home/runtime/jdk1.8.0_112/bin/java -Djava.util.logging.config.file=/home/server/apache-tomcat-7.0.6
│ ├─629c09fa97c5bb874599c162791cea17eb6e79a733527ffc8272a21817d0d20d
│ │ ├─2824 /bin/bash /usr/sbin/run-vsftpd.sh
│ │ └─3513 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
│ ├─0197de86d9acca0ebc38f62a1c75f6ccacf452fb873cf7cd96843f36412eadf6
│ │ └─2700 /home/runtime/jdk1.8.0_112/bin/java -Djava.util.logging.config.file=/home/server/apache-tomcat-7.0.6
│ ├─563def50d983b3c67e380a82234f661d59f746e7bcdf4a543fd219d49b51b556
│ │ └─2435 /home/runtime/jdk1.8.0_112/bin/java -Djava.util.logging.config.file=/home/server/apache-tomcat-7.0.6
│ └─405fca01528ab282ecaf66b4c7b197d24b273c296c37b4759d9ae470f3d729d7
│   ├─ 2366 /bin/bash /usr/bin/run-novmc.sh
│   ├─ 2846 /usr/bin/python /usr/bin/websockify --web /opt/kvm-novnc/novnc --token-plugin=TokenDB --token-sourc
│   ├─ 2847 /usr/bin/python /opt/kvm-novnc/novnc_service.py
│   └─20262 sleep 10
├─user.slice
│ └─user-0.slice
│   ├─session-3.scope
│   │ ├─3667 sshd: root@notty
│   │ └─3691 /usr/libexec/openssh/sftp-server
│   ├─session-2.scope
│   │ ├─ 3565 sshd: root@pts/0
│   │ ├─ 3717 -bash
│   │ ├─20404 systemd-cgls
│   │ └─20405 less
│   └─session-1.scope
│     ├─1373 login -- root
│     └─2128 -bash
└─system.slice
  ├─docker.service

```

### 可用的资源管理器(子系统)

- blkio 对输入/输出访问存取块设备
- cpu 配置cpu使用资源
- cpuacct 自动生成task占用的cpu报告
- cpuset 多核系统中为程序或者进程分配cpu
- devices 存取设备
- freezer 暂停或者恢复task
- net_cls 设定数据包优先级
- memory 对task进行内存使用限制
- perf_event 允许使用perf来监控cgroup
- hugetlb 允许使用大篇幅内存页

这些可用的子系统会挂载到目录`/sys/fs/cgroup`

```bash
[root@docker-pcp cgroup]# ls
blkio  cpu  cpuacct  cpu,cpuacct  cpuset  devices  freezer  hugetlb  memory  net_cls  perf_event  systemd
```

- systemd 使用程序systemd自己使用的

### 容器使用cgroup

docker进程会分别在子系统中建立group docker，当创建运行一个容器的时候，就会在docker目录内创建容器名称的group

```bash
[root@docker-pcp cgroup]# ls
blkio  cpu  cpuacct  cpu,cpuacct  cpuset  devices  freezer  hugetlb  memory  net_cls  perf_event  systemd
[root@docker-pcp cgroup]# ll cpu
lrwxrwxrwx 1 root root 11 Oct 17 10:26 cpu -> cpu,cpuacct
[root@docker-pcp cgroup]# ls cpu
cgroup.clone_children  cpuacct.stat          cpu.cfs_quota_us   cpu.stat           system.slice
cgroup.event_control   cpuacct.usage         cpu.rt_period_us   docker             tasks
cgroup.procs           cpuacct.usage_percpu  cpu.rt_runtime_us  notify_on_release  user.slice
cgroup.sane_behavior   cpu.cfs_period_us     cpu.shares         release_agent
[root@docker-pcp cgroup]# ls cpu/docker/
cfcfa2db0d749e925dee4c83510349bf4dd53e15e1eb467ffb455978c504d997  cpuacct.usage         cpu.rt_runtime_us
cgroup.clone_children                                             cpuacct.usage_percpu  cpu.shares
cgroup.event_control                                              cpu.cfs_period_us     cpu.stat
cgroup.procs                                                      cpu.cfs_quota_us      notify_on_release
cpuacct.stat                                                      cpu.rt_period_us      tasks
[root@docker-pcp cgroup]# ls cpu/docker/cfcfa2db0d749e925dee4c83510349bf4dd53e15e1eb467ffb455978c504d997/
cgroup.clone_children  cpuacct.stat          cpu.cfs_period_us  cpu.rt_runtime_us  notify_on_release
cgroup.event_control   cpuacct.usage         cpu.cfs_quota_us   cpu.shares         tasks
cgroup.procs           cpuacct.usage_percpu  cpu.rt_period_us   cpu.stat

```

在group内文件cgroup.clone_children等是对当前group和子group进行的控制参数。

### cgroup管理

推荐使用systemd来管理，当遇到管理net_prio时需要用到工具 libcgroup 工具

```bash
yum install libcgroup libcgroup-tools -y
```

也可以直接在group目录内进行文件参数管理

## Linux OverlayFS

OverlayFS是一种堆叠文件系统，他并不参与磁盘空间结构的划分，而是以其他文件系统为基础(ext4fs和xfs等)将原来底层文件的不通目录进行合并，下图merge dir目录为挂载点

![overlayfs](https://qiniu.li-rui.top/overlayfs.png)

### overlayfs特点

- 上下层同名目录合并
- 上下层同名文件覆盖
- lower dir文件写时拷贝，写时复制

### 容器使用

Linux-4.0 以后驱动

![overlayfs2](https://qiniu.li-rui.top/overlayfs2.png)

原来驱动

![overlayfs1](https://qiniu.li-rui.top/overlayfs1.png)

### 挂载一个overlay

```bash
#所需目录构建
├── lower1
│   ├── dir
│   │   ├── aa
│   │   └── bb
│   └── foo1
├── lower2
│   ├── dir
│   │   └── aa
│   └── foo2
├── merge
│   ├── dir
│   │   ├── aa
│   │   └── bb
│   ├── foo2
│   └── foo3
├── upper
│   ├── dir
│   │   └── bb
│   └── foo3
└── worker
    └── work
#创建所需目录
mkdir -p lower1 lower2 upper worker merge
mkdir -p lower1/dir lower2/dir upper/dir
#创建所需文件
touch lower1/foo1 lower2/foo2 upper/foo3
touch lower1/dir/aa lower2/dir/aa lower1/dir/bb upper/dir/bb
#写入标志性内容
echo "from lower1" > lower1/dir/aa
echo "from lower2" > lower2/dir/aa
echo "from lower1" > lower1/dir/bb
echo "from upper" > upper/dir/bb
#进行挂载
mount -t overlay overlay -o lowerdir=./lower1,lowerdir=lower2,upperdir=./upper,workdir=./worker ./merge

```

## 虚拟网卡

网络虚拟化就是在一台机器上分离出多个网络协议栈。Linux中最直接的就是通过net namespace来实现。

下面会介绍docker会用到的虚拟网卡技术

### VETH

必须成对出现

- 到外veth外部网络，通过对其中一个veth绑定网桥实现
- veth之间互通，用来在net namespace之间传递数据包

![veth](https://qiniu.li-rui.top/veth.png)

配置veth

```bash
ip netns add ns1
ip link add v1 type veth peer name veth1
ip link set v1 netns ns1
brctl addbr br0
brctl addif br0 eth0
brctl addif br0 veth1
ifconfig br0 192.168.0.1/16

```

### MACVLAN

macvalan将一块物理网卡，虚拟出来多块网卡，每块网卡和物理网卡的mac地址和IP都不一样

可以通过命令创建macvlan
```bash
ip link add link eth0 name macv1 type macvlan 
#指定模式
ip link add link eth0 name macv1 type macvlan mode bridge|vepa|private
```

![macvlan](https://qiniu.li-rui.top/macvlan.png)

macvlan存在多种运行模式

#### bridge

这个模式下，物理机网卡和macvlan网卡均可以直接进行数据通信，不需要转发，二层网络

![macvlan-briage](https://qiniu.li-rui.top/macvlan-briage.png)

#### VEPA

在这个模式下，macvlan网卡之间并不可以直接通信，因为没有内建桥，需要宿主机网卡链接的交换机来控制数据包转发，开启发夹弯模式就是数据包180度大转弯模式

![macvlan-vepa](https://qiniu.li-rui.top/macvlan-vepa.png)

#### private

private模式是vepa的隔离性进一步增强，即便外部交换机开启了发夹弯模式，链接到同一个网卡的macvlan网卡也无法接收到对方的广播包

![macvlan-private](https://qiniu.li-rui.top/macvlan-private.png)

#### 配置macvaln

```bash
ip link add link eth0 name macv1 type macvlan
ip link set macv1 netns ns1
```

相对于使用桥，macvlan更直接，省掉了寻找哈希表的过程

#### 为什么不推荐内部交换机

![why](https://qiniu.li-rui.top/why.png)

![mav](https://qiniu.li-rui.top/mav.png)

宿主机上可能拥有一块以太网卡，但是从该网卡发出的数据包却不一定来自同一个协议栈，它可能来自不同的虚拟机或者不同的net namespace(仅针对Linux)

### IPVLAN

IPVLAN也是一种虚拟网卡技术，宿主机虚拟出来的网卡每一块的mac都一样，IP不一样

![ipvlan](https://qiniu.li-rui.top/ipvlan.jpg)

创建ipvlan

```bash
ip link add link <master-dev> <slave-dev> type ipvlan mode { l2 | L3 }
```

#### l2/l3/macvlan

l2和macvlan bridge模式类似

![l2](https://qiniu.li-rui.top/l2.png)

l3相当于宿主机网卡为三层交换机，不通的ipvlan网卡可以配置不同的网段

![l3](https://qiniu.li-rui.top/l3.png)

ipvlan和macvlan不可以同时使用对同一个宿主机网卡，无线只可以ipvlan，交换机限制mac数量可以使用ipvlan

### MacVTap

MacVTap是基于macvlan模块的，因此也持macvlan的三种模式:`Bridge`,`VEPA`,`private`,每次创建一个macvtap设备必定会创建一个macvlan设备，主要作用是将macvlan设备获取的数据包直接上送到用户态而不是协议栈

macvtap 设备收到的数据包直接上送到用户态，用户态程序通过macvtap设备关联的字符设备直接发送数据包。字符设备：提供连续的数据流，应用程序可以顺序读取

MacVTap是用来替代TUN/TAP和Bridge内核模块的

![mactap](https://qiniu.li-rui.top/mactap.png)
















































