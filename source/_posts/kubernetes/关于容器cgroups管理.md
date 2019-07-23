---
title: 关于容器cgroups管理
date: 2019-07-23 12:12:28
tags:
- kubernetes
---

# cgroup

cgroup用来对进行资源限制，在Linux上就是一个文件系统，默认被systemd挂载到`/sys/fs/cgroup`

<!--more-->

cgroup的管理现在有两种

- cgroupfs 就是一个文件系统，通过编辑文件来更改
- systemd 提供了更改的接口，实质是对cgroupfs做了封装

## docker中的cgroup

### 默认使用cgroupfs

docker中默认使用cgroups，[详见](https://docs.docker.com/v17.12/engine/reference/commandline/dockerd/#docker-runtime-execution-options)

```
The native.cgroupdriver option specifies the management of the container’s cgroups. You can only specify cgroupfs or systemd. If you specify systemd and it is not available, the system errors out. If you omit the native.cgroupdriver option, cgroupfs is used.
```

### 使用systemd

```json
///etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```

## kubelet中cgroup

默认也是cgroupfs

```bash
--cgroup-driver string Driver that the kubelet uses to manipulate cgroups on the host.  Possible values: 'cgroupfs', 'systemd' (default "cgroupfs")
```

# 官方推荐

使用systemd，[详见](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

```bash
Control groups are used to constrain resources that are allocated to processes. A single cgroup manager will simplify the view of what resources are being allocated and will by default have a more consistent view of the available and in-use resources. When we have two managers we end up with two views of those resources. We have seen cases in the field where nodes that are configured to use cgroupfs for the kubelet and Docker, and systemd for the rest of the processes running on the node becomes unstable under resource pressure.
```

