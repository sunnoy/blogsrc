---
title: pod调试工具kubectl-debug
date: 2019-09-20 12:12:28
tags:
- kubernetes
---

# 容器调试的问题

容器内不都是安装了调试工具，因此可以利用namespace的机制来在宿主机上来做操作

<!--more-->

# kubectl-debug

kubectl-debug通过代理的方式可以在本地去调试远程集群，而不用去远程node上登陆去调试

## 具体过程

![config](https://qiniu.li-rui.top/config.png)

- 插件查询 ApiServer：demo-pod 是否存在，所在节点是什么
- ApiServer 返回 demo-pod 所在所在节点
- 插件请求在目标节点上创建 Debug Agent Pod
- Kubelet 创建 Debug Agent Pod
- 插件发现 Debug Agent 已经 Ready，发起 debug 请求（长连接）
- Debug Agent 收到 debug 请求，创建 Debug 容器并加入目标容器的各个 Namespace 中，创建完成后，与 Debug 容器的 tty 建立连接
- 接下来，客户端就可以开始通过 5，6 这两个连接开始 debug 操作。操作结束后，Debug Agent 清理 Debug 容器，插件清理 Debug Agent，一次 Debug 完成。

可见远程node上会安装两个容器，agent容器和本地客户端相连，debug容器和目标容器在同一个namespace。本地只需要安装个命令行就行了。

debhug容器默认使用nicolaka/netshoot镜像，也可以自定义debug镜像

## 安装

```bash
export PLUGIN_VERSION=0.1.1
# linux x86_64
curl -Lo kubectl-debug.tar.gz https://github.com/aylei/kubectl-debug/releases/download/v${PLUGIN_VERSION}/kubectl-debug_${PLUGIN_VERSION}_linux_amd64.tar.gz
# macos
curl -Lo kubectl-debug.tar.gz https://github.com/aylei/kubectl-debug/releases/download/v${PLUGIN_VERSION}/kubectl-debug_${PLUGIN_VERSION}_darwin_amd64.tar.gz

tar -zxvf kubectl-debug.tar.gz kubectl-debug
sudo mv kubectl-debug /usr/local/bin/

```

通过将kubectl-debug移至`PATH`变量，就可以通过命令`kubectl debug`来操作

## 使用

使用方式分为两种，一种是安装agent容器到每个node上，这样就可以加快启动速度。本文没有采用这种方式，因此需要添加`-a`参数

```bash
kubectl debug -h

Run a container in a running pod, this container will join the namespaces of an existing container of the pod.

You may set default configuration such as image and command in the config file, which locates in "~/.kube/debug-config" by default.

Usage:
  debug POD [-c CONTAINER] -- COMMAND [args...]

Examples:

	# debug a container in the running pod, the first container will be picked by default
	kubectl debug POD_NAME

	# specify namespace or container
	kubectl debug --namespace foo POD_NAME -c CONTAINER_NAME

	# override the default troubleshooting image
	kubectl debug POD_NAME --image aylei/debug-jvm

	# override entrypoint of debug container
	kubectl debug POD_NAME --image aylei/debug-jvm /bin/bash

	# override the debug config file
	kubectl debug POD_NAME --debug-config ./debug-config.yml

	# check version
	kubectl --version


Flags:
      --agent-image string             Agentless mode, the container Image to run the agent container , default to aylei/debug-agent:latest
      --agent-pod-name-prefix string   Agentless mode, pod name prefix , default to debug-agent-pod
      --agent-pod-namespace string     Agentless mode, agent pod namespace, default to default
      #集群内没有安装agent需要使用这个 -a 选项
 -a, --agentless                      Whether to turn on agentless mode. Agentless mode: debug target pod if there isn't an agent running on the target host.
      --as string                      Username to impersonate for the operation
      --as-group stringArray           Group to impersonate for the operation, this flag can be repeated to specify multiple groups.
      --cache-dir string               Default HTTP cache directory (default "/root/.kube/http-cache")
      --certificate-authority string   Path to a cert file for the certificate authority
      --client-certificate string      Path to a client certificate file for TLS
      --client-key string              Path to a client key file for TLS
      --cluster string                 The name of the kubeconfig cluster to use
  -c, --container string               Target container to debug, default to the first container in pod
      --context string                 The name of the kubeconfig context to use
      --daemonset-name string          Debug agent daemonset name when using port-forward
      --daemonset-ns string            Debug agent namespace, default to 'default'
      --debug-config string            Debug config file, default to ~/.kube/debug-config
      --fork                           Fork a new pod for debugging (useful if the pod status is CrashLoopBackoff)
  -h, --help                           help for debug
      --image string                   Container Image to run the debug container, default to nicolaka/netshoot:latest
      --insecure-skip-tls-verify       If true, the server's certificate will not be checked for validity. This will make your HTTPS connections insecure
      --kubeconfig string              Path to the kubeconfig file to use for CLI requests.
  -n, --namespace string               If present, the namespace scope for this CLI request
  -p, --port int                       Agent port for debug cli to connect, default to 10027
      --port-forward                   Whether using port-forward to connect debug-agent
      --request-timeout string         The length of time to wait before giving up on a single server request. Non-zero values should contain a corresponding time unit (e.g. 1s, 2m, 3h). A value of zero means don't timeout requests. (default "0")
  -s, --server string                  The address and port of the Kubernetes API server
      --token string                   Bearer token for authentication to the API server
      --user string                    The name of the kubeconfig user to use
      --version                        version for debug
```

## 正在运行的pod

```bash
kubectl debug -n cattle-system cattle-cluster-agent-6594d48478-44z24  -a

Agent Pod info: [Name:debug-agent-pod-9c84779a-db79-11e9-8836-acde48001122, Namespace:default, Image:aylei/debug-agent:latest, HostPort:10027, ContainerPort:10027]
Waiting for pod debug-agent-pod-9c84779a-db79-11e9-8836-acde48001122 to run...
pulling image nicolaka/netshoot:latest...
latest: Pulling from nicolaka/netshoot
Digest: sha256:8b020dc72d8ef07663e44c449f1294fc47c81a10ef5303dc8c2d9635e8ca22b1
Status: Image is up to date for nicolaka/netshoot:latest
starting debug container...
container created, open tty...
bash-5.0# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if38: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether 6e:0b:b5:b5:12:ad brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.20.0.32/24 scope global eth0
       valid_lft forever preferred_lft forever
```

## 进入容器文件系统

```bash
chroot /proc/1/root

groups: cannot find name for group ID 11
root@cattle-cluster-agent-6594d48478-44z24:/# ls
bin  boot  cattle-credentials  connected  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

## CrashLoopBackoff容器

基本方式是重新运行一个容器，但是容器内的执行命令就会更换为sh，类似

```bash
docker run -it --rm centos:7 sh
```
在kubectl-debug中使用参数`--fork`来实现

```bash
kubectl debug -n cattle-system cattle-cluster-agent-6594d48478-44z24  -a --fork

Agent Pod info: [Name:debug-agent-pod-a8bdbfd4-db7a-11e9-8a8b-acde48001122, Namespace:default, Image:aylei/debug-agent:latest, HostPort:10027, ContainerPort:10027]
Waiting for pod debug-agent-pod-a8bdbfd4-db7a-11e9-8a8b-acde48001122 to run...
Waiting for pod cattle-cluster-agent-6594d48478-44z24-ab0c16e6-db7a-11e9-8a8b-acde48001122-debug to run...
pulling image nicolaka/netshoot:latest...
latest: Pulling from nicolaka/netshoot
Digest: sha256:8b020dc72d8ef07663e44c449f1294fc47c81a10ef5303dc8c2d9635e8ca22b1
Status: Image is up to date for nicolaka/netshoot:latest
starting debug container...
container created, open tty...

#查看pod
k get pod -n cattle-system
NAME                                                                               READY   STATUS    RESTARTS   AGE
cattle-cluster-agent-6594d48478-44z24                                              1/1     Running   0          20h
cattle-cluster-agent-6594d48478-44z24-ab0c16e6-db7a-11e9-8a8b-acde48001122-debug   1/1     Running   0          17s
```

## 自定义debug镜像

```bash
# sh 为在debug容器启动后执行的命令
kubectl debug -n cattle-system cattle-cluster-agent-6594d48478-44z24  -a --image busybox sh
```