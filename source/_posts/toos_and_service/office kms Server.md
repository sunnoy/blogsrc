---
title: office kms Server
date: 2018-11-05 12:12:28
tags:
- kms
---
# office kms Server

## office

### 安装

```bash
docker run --restart=always -p 1030:1688 --name office -d luodaoyi/kms-server
```
<!--more-->

### 使用

#### KMS激活使用方法

一般来说，只要确保的下载的是VL批量版本并且没有手动安装过任何key，
你只需要使用管理员权限运行cmd执行一句命令就足够：

```
slmgr /skms kms.luody.info
```


然后去计算机属性或者控制面板其他的什么的地方点一下激活就好了。

当然，如果你懒得点，可以多打一句命令手动激活：

```
slmgr /ato
```

这句命令的意思是，马上对当前设置的key和服务器地址等进行尝试激活操作。

kms激活的前提是你的系统是批量授权版本，即VL版，一般企业版都是VL版，专业版有零售和VL版，家庭版旗舰版OEM版等等那就肯定不能用kms激活。一般建议从[http://msdn.itellyou.cn](http://msdn.itellyou.cn)上面下载系统
VL版本的镜像一般内置GVLK key，用于kms激活。如果你手动输过其他key，那么这个内置的key就会被替换掉，这个时候如果你想用kms，那么就需要把GVLK key输回去。首先，
到[https://technet.microsoft.com/en-us/library/jj612867.aspx](https://technet.microsoft.com/en-us/library/jj612867.aspx)
获取你对应版本的KEY
如果打不开下面有对应的

如果不知道自己的系统是什么版本，可以运行以下命令查看系统版本：

```
wmic os get caption
```

得到对应key之后，使用管理员权限运行cmd执行安装key：

```
slmgr /ipk xxxxx-xxxxx-xxxxx-xxxxx
```
然后跟上面说的一样设置kms服务器地址，激活。


#### 一句命令激活OFFICE

首先你的OFFICE必须是VOL版本，否则无法激活。
找到你的office安装目录，比如

```
C:\Program Files (x86)\Microsoft Office\Office16
```

64位的就是

```
C:\Program Files\Microsoft Office\Office16
```

office16是office2016，office15就是2013，office14就是2022.

然后目录对的话，该目录下面应该有个OSPP.VBS。

接下来我们就cd到这个目录下面，例如：

```
cd C:\Program Files (x86)\Microsoft Office\Office16
```

然后执行注册kms服务器地址：

```
cscript ospp.vbs /sethst:kms.luody.info
```
/sethst参数就是指定kms服务器地址。

一般ospp.vbs可以拖进去cmd窗口，所以也可以这么弄：

```
cscript "C:\Program Files (x86)\Microsoft Office\Office16\OSPP.VBS" /sethst:kms.luody.info
```

一般来说，“一句命令已经完成了”，但一般office不会马上连接kms服务器进行激活，所以我们额外补充一条手动激活命令：

```
cscript ospp.vbs /act
```


如果提示看到successful的字样，那么就是激活成功了，重新打开office就好。

