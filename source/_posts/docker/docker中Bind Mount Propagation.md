---
title: docker中Bind Mount Propagation
date: 2019-02-09 12:12:28
tags:
- docker
---

# docker中数据持久化

docker为我们提供了三种持久化类型，具体见下图：

![types-of-mounts](https://qiniu.li-rui.top/types-of-mounts.png)
<!--more-->

- Volumes是被docker进程管理的
- Bind mounts用于宿主机和docker容器之间进行文件共享
- tmpfs用于容器使用宿主机部分内存

数据的持久化本质上是不同文件目录的mount，是mount就涉及到mount传播(Mount Propagation)，mount传播主要在Bind mounts中配置。volumes中mount传播配置默认为rprivate，但是不提供其他类型的配置。

本文主要讨论mount传播的内核实现以及相关配置

# namespace

## nsproxy

我们知道，Linux使用namespace来对系统资源进行隔离。不同namespace中的进程看到的资源和进程表是不一样的，同一namespace中的不同进程看到的资源和进程表确是一样的。

Linux内核中结构体nsproxy来管理namespace

```c
struct nsproxy {
	atomic_t count;               /* 被引用的次数 */
	struct uts_namespace *uts_ns; /* 主机名和域名 */
	struct ipc_namespace *ipc_ns; /* 进程通信 */
	struct mnt_namespace *mnt_ns; /* mount */
	struct pid_namespace *pid_ns; /* pid */
	struct net 	     *net_ns;     /* 网络 */
};
```

## mnt namespace

### 创建
新的mnt namespace将会被调用函数clone()创建，该函数用来创建新的进程，我们来看一下该函数额传参

```c
int clone(int (*fn)(void * arg), void *stack, int flags, void * arg);
```

fn：指向程序的指针
child_stack：为子进程分配的堆栈空间
flags：描述出从父进程所继承的资源，当为CLONE_NEWNS就会拷贝父进程的mnt namespace
arg：传给子进程的参数

### 数据结构

mnt_namespace中主要看结构体vfsmount

```c 
struct mnt_namespace {
	atomic_t		count;
	struct vfsmount *	root;///当前namespace下的根文件系统
	struct list_head	list; ///当前namespace下的文件系统链表（vfsmount list）
	wait_queue_head_t poll;
	int event;
};
```

vfsmount结构体主要存放当前跟文件系统的信息，其中dentry(Directory entry)用来记录目录的信息，比如/usr/local/lib中有 / usr local lib 4个dentry

```c
struct vfsmount {
        struct list_head mnt_hash;
        struct vfsmount *mnt_parent;    /* fs we are mounted on */
        struct dentry *mnt_mountpoint;  /* dentry of mountpoint */
        struct dentry *mnt_root;        /* root of the mounted
                                           tree*/
        struct super_block *mnt_sb;     /* pointer to superblock */
        struct list_head mnt_mounts;    /* list of children,
                                           anchored here */
        struct list_head mnt_child;     /* and going through their
                                           mnt_child */
        atomic_t mnt_count;
        int mnt_flags;
        char *mnt_devname;              /* Name of device e.g. 
                                           /dev/dsk/hda1 */
        struct list_head mnt_list;
};
```

dentry数据结构如下

```c
struct dentry {
        struct inode *d_inode;
        struct dentry *d_parent;
        struct qstr d_name;
        struct list_head d_subdirs;  /* sub-directories/files     */
        struct list_head d_child;    /* sibling directories/files */
        ...
}
```

# bind mount

bind mount是一项内核的特性，可以将一个目录挂载到另一个目录，可以理解是ln对目录的硬链接，ln对文件有硬链接和软链接，但是ln对目录只有软链接。

## 挂载示例

准备两个目录并在里面创建文件

```bash
mkdir src
mkdir targ
echo "src" > src/src
echo "target" > targ/targ
#记下目录的inode号
ls -lid src

34403169 drwxr-xr-x 2 root root 17 Dec 26 14:09 src
ls -lid targ/
67149570 drwxr-xr-x 2 root root 18 Dec 26 14:09 targ/
```

执行挂载并查看，可见操作会将src的inode号挂载到targ上面，也就是说目录src和targ同时链接到src的inode 34403169上面

```bash
mount --bind /root/src/ /root/targ/

ls -lid src
34403169 drwxr-xr-x 2 root root 17 Dec 26 14:09 src
ls -lid targ/
34403169 drwxr-xr-x 2 root root 17 Dec 26 14:09 targ/
```

取消挂载后，targ依然指向原来的inode，targ中原来的文件并不会收到影响

```bash
umount /root/targ

ls -lid src
34403169 drwxr-xr-x 2 root root 17 Dec 26 15:15 src

ls -lid targ/
67149570 drwxr-xr-x 2 root root 18 Dec 26 14:09 targ/
```

## 内核层面

创建一个bind mount就会生成一个vfsmount以及里面的dentry

```c
///传入参数为source对应的vfsmount、dentry，返回新的vfsmount
static struct vfsmount *
clone_mnt(struct vfsmount *old, struct dentry *root)
{
	struct super_block *sb = old->mnt_sb;
	struct vfsmount *mnt = alloc_vfsmnt(old->mnt_devname); ///source的device name

	if (mnt) {
		mnt->mnt_flags = old->mnt_flags;
		atomic_inc(&sb->s_active);
		mnt->mnt_sb = sb;
		mnt->mnt_root = dget(root); ///本文件系统的根目录
		mnt->mnt_mountpoint = mnt->mnt_root; ///挂载点暂时指向本文件系统的根目录,后面再修改指向target的挂载点目录
		mnt->mnt_parent = mnt; ///暂时指向自己
		mnt->mnt_namespace = old->mnt_namespace;
	}
	return mnt;
}
```

也可以通过`cat /proc/self/mountinfo`查看
```bash
#当前mount id为217，父mount id为62
217 62 253:0 /root/src /root/targ rw,relatime shared:1 - xfs /dev/mapper/centos-root rw,attr2,inode64,noquota

62 1 253:0 / / rw,relatime shared:1 - xfs /dev/mapper/centos-root rw,attr2,inode64,noquota
```

# 容器应用

## 容器创建

创建一个容器的同时就会创建vfsmount树

```bash
# 创建容器之前系统中
                  A (/)
                /   \
       (/proc) B     C (/tmp)
# 执行创建容器命令
docker run -d -it ubuntu
# 创建容器后
                  A (/)                 # 宿主机的 vfsmount tree
                /   \
       (/proc) B     C (/tmp)
--------------------------------------------
                  A'(/)                 # 容器的 vfsmount tree
                /   \
      (/proc) B'     C' (/tmp)
```

## 进行bind mount操作

执行bind mount操作就是在容器内的vfsmount树中新建一个dentry，并将该dentry执行宿主机中的dentry，也就是说这两个dentry是一样的

```bash
# 创建容器并进行bind mount操作
docker run -d -it -v /tmp/path:/tmp/path ubuntu
--------------------------------------------------------------
   Host vfmount tree           |  Container's vfsmount tree
-------------------------------------------------------------- 
       A (/)                   |                A'(/)
         \                     |                  \
          C (/tmp)             |                   C'(/tmp)
          |                    |                   |
          '--> (dentry /tmp)   |   (dentry /tmp) <-'
                |              |           |
                '---> (dentry /path)  <----'
```

## 考虑mount传播

那么，此时在宿主机上执行bind mount操作到/tmp/path，/tmp/path/some/path会在容器内部出现么？

```bash
# 创建容器
docker run -d -it -v /tmp/path:/tmp/path ubuntu
--------------------------------------------------------------
   Host vfmount tree           |  Container's vfsmount tree
-------------------------------------------------------------- 
       A (/)                   |                A'(/)
         \                     |                  \
          C (/tmp)             |                   C'(/tmp)
          |                    |                   |
          '--> (dentry /tmp)   |   (dentry /tmp) <-'
                |              |           |
                '---> (dentry /path)  <----'
# 宿主机上执行bind mount操作
mount --bind /some/path /tmp/path/some/path
```

答案是看不到，因为docker中bind mount传播默认是`rprivate`，也就是说宿主机和容器内部vfsmount树并没有共享，尽管他们某一个dentry共享了。

Linux内核提供了许多vfsmount树共享的模式

- Shared bind-mount
  - shared 
  - rshared
- Non-Shared bind-mount
  - slave
  - rslave 
  - unbindable

### Shared bind-mount

Shared bind-mount就是双向传播，不管从宿主机到容器，还是从容器到宿主机。

rshared 表示递归分享

### Non-Shared Bind Mount

Non-Shared Bind Mount包含集中模式

Slave, Rslave 表示mount事件可以从宿主机到容器，但是不能从容器到宿主机

Private, rprivate 表示mount事件禁止传播

Unbindable 该模式不适用容器

### 加入传播选项

下面就会在容器内看到

```bash
# 创建容器
docker run -d -it -v /tmp/path:/tmp/path:rshared ubuntu
--------------------------------------------------------------
   Host vfmount tree           |  Container's vfsmount tree
-------------------------------------------------------------- 
       A (/)                   |                A'(/)
         \                     |                  \
          C (/tmp)             |                   C'(/tmp)
          |                    |                   |
          '--> (dentry /tmp)   |   (dentry /tmp) <-'
                |              |           |
                '---> (dentry /path)  <----'
# 宿主机上执行bind mount操作
mount --bind /some/path /tmp/path/some/path
```

docker提供传播选项总结，[来源](https://docs.docker.com/storage/bind-mounts/#configure-bind-propagation)

![mount](https://qiniu.li-rui.top/mount.png)

# Kubernetes

Kubernetes提供了两种模式

- Bidirectional 和rshared一样
- HostToContainer 和rslave一样，默认模式

## 使用示例

```yml
deployment:
  containers:
    - image: centos
      name: test
      volume:
        - mount: /usr/test-pod
          store: local-vol
          #这里设置双向传播
          propagation: bidirectional
  name: local-test-reader
  version: extensions/v1beta1
  volumes:
    local-vol: pvc:example-local-claim
```










