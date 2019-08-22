---
title: oracle-jdk8容器资源限制测试
date: 2019-08-22 12:12:28
tags:
- docker
---

# jdk与docker

默认jvm并不知道自己运行在容器中，也就是说jvm获取内存是通过文件/proc/meminfo来的，而容器内的这个文件和宿主机上的一致

<!--more-->

# 结论先行

无论是oracle jdk镜像 1.8.0_211 还是 oracle jdk包 1.8.0_221 都已经可以识别容器内存限制而去计算jvm的 Default Heap Size

# 测试环境

- 宿主机内存8g
- 宿主机cpu 4核
- oracle jdk镜像 1.8.0_211
- oracle jdk包 1.8.0_221

# oracle官方docker镜像测试

地址在 https://hub.docker.com/_/oracle-serverjre-8 需要订阅，这里pull后上传到docker hub，现在可以直接拉取sunnoy/oraclejdk:8

本文主要测试内存

## 不加任何限制

### MaxRAMFraction

JVM的Max Heap Size是系统内存的1/4，假如我们系统是8G，那么JVM将的默认Heap≈2G。大约是识别内存（-XX:MaxRAM）的四分之一，这个比例是通过参数-XX:MaxRAMFraction=int来设定的，默认为MaxRAMFraction=4，也就是说MaxRAM用来计算 Default Heap Size

```bash
# 默认情况
# 查看
docker run --rm --name jdk -it  sunnoy/oraclejdk:8 java -XX:+PrintFlagsFinal -version | grep -Ei "maxheapsize|maxram"

    uintx DefaultMaxRAMFraction                     = 4                                   {product}
    uintx MaxHeapSize                              := 2051014656                          {product}
 uint64_t MaxRAM                                    = 137438953472                        {pd product}
    uintx MaxRAMFraction                            = 4                                   {product}
   double MaxRAMPercentage                          = 25.000000                           {product}
2051014656/1024^3 = 1.91g

# 通过-XshowSettings:vm查看相差不大
docker run --rm --name jdk -it  sunnoy/oraclejdk:8 java -XshowSettings:vm  -version

VM settings:
    Max. Heap Size (Estimated): 1.70G
    Ergonomics Machine Class: server
    Using VM: Java HotSpot(TM) 64-Bit Server VM

java version "1.8.0_211"
Java(TM) SE Runtime Environment (build 1.8.0_211-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.211-b12, mixed mode)



# MaxRAMFraction=1
docker run --rm --name jdk -it  sunnoy/oraclejdk:8 java -XX:MaxRAMFraction=1 -XshowSettings:vm  -version

VM settings:
    Max. Heap Size (Estimated): 6.79G


# MaxRAMFraction=2
docker run --rm --name jdk -it  sunnoy/oraclejdk:8 java -XX:MaxRAMFraction=2 -XshowSettings:vm  -version

VM settings:
    Max. Heap Size (Estimated): 3.40G


# MaxRAMFraction=3
docker run --rm --name jdk -it  sunnoy/oraclejdk:8 java -XX:MaxRAMFraction=3 -XshowSettings:vm  -version

VM settings:
    Max. Heap Size (Estimated): 2.26G
```

详细的比例关系

```bash

```
+----------------+-------------------+
| MaxRAMFraction | % of RAM for heap |
|----------------+-------------------|
|              1 |              100% |
|              2 |               50% |
|              3 |               33% |
|              4 |               25% |
+----------------+-------------------+

## 添加内存限制

```bash
docker run --rm --name jdk -it -m 400m  sunnoy/oraclejdk:8 java -XshowSettings:vm  -version

VM settings:
    Max. Heap Size (Estimated): 121.81M

# MaxRAMFraction=1
docker run --rm --name jdk -it -m 400m  sunnoy/oraclejdk:8 java -XX:MaxRAMFraction=1 -XshowSettings:vm  -version

VM settings:
    Max. Heap Size (Estimated): 386.69M

```

## 增加实验特性

- -XX:+UnlockExperimentalVMOptions
- -XX:+UseCGroupMemoryLimitForHeap 使用的前提是开启UnlockExperimentalVMOptions

```bash
docker run --rm --name jdk -it -m 400m  sunnoy/oraclejdk:8 java -XX:+UnlockExperimentalVMOptions  -XX:+UseCGroupMemoryLimitForHeap -XshowSettings:vm  -version

VM settings:
    Max. Heap Size (Estimated): 121.81M
```

**可见oracle官方的docker镜像已经默认开启实验特性，其实是jvm识别容器中的/sys/fs/cgroup/memory/memory.limit_in_bytes去替代-XX:MaxRAM的值**

```bash
docker run --rm --name jdk -it -m 400m  sunnoy/oraclejdk:8 cat /sys/fs/cgroup/memory/memory.limit_in_bytes

419430400

docker run --rm --name jdk -it -m 400m  sunnoy/oraclejdk:8 sh -c "java -XX:+PrintFlagsFinal -XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -version | grep -Ei 'maxheapsize|maxram'"

    uintx DefaultMaxRAMFraction                     = 4                                   {product}
    uintx MaxHeapSize                              := 132120576                           {product} 419430400/4
 uint64_t MaxRAM                                    = 137438953472                        {pd product}
    uintx MaxRAMFraction                            = 4                                   {product}
   double MaxRAMPercentage                          = 25.000000                           {product}
```

# Oracle官方jdk包测试

官方下载真是太慢了，已经转存百度云

```bash
链接:https://pan.baidu.com/s/1pQpCBjb5ad14ZpBilWVLug  密码:s81k
```

## 镜像准备

```Dockerfile
FROM centos:7

WORKDIR /home

RUN cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/shanghai" > /etc/timezone && \
    localedef -c -f UTF-8 -i zh_CN zh_CN.utf8

ADD jdk-8u221-linux-x64.tar.gz .


ENV LC_ALL "zh_CN.UTF-8"
ENV JAVA_HOME /home/jdk1.8.0_221
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV PATH $PATH:$JAVA_HOME/bin

CMD ["java", "-version"]
```

已经上传至docker hub可以直接拉取sunnoy/jdk-d:8

## 不加任何限制

```bash
docker run --rm --name jdk -it sunnoy/jdk-d:8 java -XshowSettings:vm  -version

VM settings:
    Max. Heap Size (Estimated): 1.70G
    Ergonomics Machine Class: server
    Using VM: Java HotSpot(TM) 64-Bit Server VM

java version "1.8.0_221"
Java(TM) SE Runtime Environment (build 1.8.0_221-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.221-b11, mixed mode)
```

## 添加内存限制

```bash
## 仅仅添加内存限制
docker run --rm --name jdk -it -m 400m sunnoy/jdk-d:8 java -XshowSettings:vm  -version

VM settings:
    Max. Heap Size (Estimated): 121.81M

## 加入内存限制 加入实验参数 
docker run --rm --name jdk -it -m 400m sunnoy/jdk-d:8 java -XX:+UnlockExperimentalVMOptions  -XX:+UseCGroupMemoryLimitForHeap -XshowSettings:vm  -version

VM settings:
    Max. Heap Size (Estimated): 121.81M

## 加入内存限制 加入实验参数 MaxRAMFraction=1
docker run --rm --name jdk -it -m 400m sunnoy/jdk-d:8 java -XX:MaxRAMFraction=1 -XX:+UnlockExperimentalVMOptions  -XX:+UseCGroupMemoryLimitForHeap -XshowSettings:vm  -version

VM settings:
    Max. Heap Size (Estimated): 386.69M

## 加入内存限制 MaxRAMFraction=1
docker run --rm --name jdk -it -m 400m sunnoy/jdk-d:8 java -XX:MaxRAMFraction=1 -XshowSettings:vm  -version

VM settings:
    Max. Heap Size (Estimated): 386.69M
```

可见，即便是直接下载的jdk 1.8.0_221也已经开启相关特性参数


## java10+如何做？

废弃掉XX:+UseCGroupMemoryLimitForHeap，引入XX:+UseContainerSupport(默认开启)和XX:MaxRAMPercentage

也就是说到了java10只需要配置XX:MaxRAMPercentage即可

## jdk8131+ jdk9 进一步优化

既然试验性参数是对MaxRAM的替代，那么就通过设置MaxRAM为memory.limit_in_bytes的一部分比如百分之70达到jdk10的效果

```bash
# sunnoy/oraclejdk:8
docker run --rm --name jdk -it -m 400m  sunnoy/oraclejdk:8 sh -c 'exec java -XX:MaxRAM=$(( $(cat /sys/fs/cgroup/memory/memory.limit_in_bytes) * 70 / 100 )) -XshowSettings:vm  -version'

VM settings:
    Max. Heap Size (Estimated): 121.81M


# sunnoy/jdk-d:8
docker run --rm --name jdk -it -m 400m  sunnoy/jdk-d:8 sh -c 'exec java -XX:MaxRAM=$(( $(cat /sys/fs/cgroup/memory/memory.limit_in_bytes) * 70 / 100 )) -XshowSettings:vm  -version'

VM settings:
    Max. Heap Size (Estimated): 121.81M
```

该优化在生产环境之前仍需要进一步测试

# k8s测试

node配置为4核8g，并且该namespace没有LimitRange

```bash
#不做任何限制
kubectl run jdk -it --rm --image=sunnoy/jdk-d:8  -- bash

java -XshowSettings:vm  -version

VM settings:
    Max. Heap Size (Estimated): 1.85G


# 做资源限制
kubectl run jdk -it --rm --image=sunnoy/jdk-d:8 --limits='cpu=200m,memory=400Mi' --requests='cpu=200m,memory=400Mi' -- bash

java -XshowSettings:vm  -version

VM settings:
    Max. Heap Size (Estimated): 121.81M
```

# 参考资料

```bash
#推荐下面这篇
https://medium.com/adorsys/jvm-memory-settings-in-a-container-environment-64b0840e1d9e

https://qingmu.io/2018/12/17/How-to-securely-limit-JVM-resources-in-a-container/
https://github.com/moby/moby/issues/15020
https://stackoverflow.com/questions/27816210/what-does-maxram-jvm-parameter-signify
https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#memory_heap
```


Oracle Java 8 SE官方镜像readme页面

>Support for docker limits on the number of CPUs
Starting with version 8u131 the Java Runtime supports Docker limits on the number of CPUs to use.
The JVM will apply the Docker --cpuset-cpus limit as the number of CPUs the JVM sees in the docker container.
If -XX:ParallelGCThreads or -XX:CICompilerCount are specified and a --cpuset-cpus limit is specified the JVM will use the -XX:ParallelGCThreads and -XX:CICompilerCount values.
See JDK-6515172 for details.
Experimental support for docker memory limits
To configure the maximum Java heap size to match Docker memory limits without setting a maximum Java heap via -Xmx use two JVM command line options:
-XX:+UnlockExperimentalVMOptions
-XX:+UseCGroupMemoryLimitForHeap.
When these two JVM command line options are used, and -Xmx is not specified, the JVM will look at the Linux cgroup configuration, which is what Docker containers use for setting memory limits, to transparently size a maximum Java heap size.
See JDK-8170888 for details.