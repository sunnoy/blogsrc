---
title: Linux找到盘符对应插槽
date: 2018-11-08 08:12:28
tags:
- Linux
---

# 磁盘盘符查找插槽

在服务器里面，磁盘插在磁盘槽内，然后通过raid或者直接向系统提供块设备

**本文内容暂时仅仅适用于使用raid卡且raid卡是LSI公司的情况**

## raid卡

### 厂商介绍

raid主要有LSI、Adaptec、Highpoint、Promise四家提供，其中前两家提供的最多.LSI现在属于博通公司。

>讲起博通的发家史，真的“一匹布那么长”，因为这家公司见证了太多半导体芯片公司的并购，都是它一手促成。最早是1961年惠普的元器件部门，并且在1999年将医疗和测试部门拆分出来，成立了Agilent安捷伦公司，元器件部门顺利成章地成为安捷伦旗下的半导体事业部，时隔六年再次将半导体事业部拆分卖出，更名为Avago安华高，并在2013年66亿美金收购了LSI艾萨华公司（LSI来历就不多说了，都是比较有名的半导体公司）。而安华高最后在2015年与博通结缘，“聘礼”高达370亿美元，这个博通就是目前的Broadcom Limited。[来源](http://www.expreview.com/57734.html)
<!--more-->

### 基本概念

#### 磁盘组

磁盘组是一组物理磁盘组成的集合，一般表述为“Drive Group”（简称“DG”）

#### 虚拟磁盘

在磁盘组的基础上划分出一组具有连续的存储存储单元，相当于一个物理磁盘。一个虚拟磁盘可以是一个磁盘组构成，也可以是几块硬盘中的一部分。虚拟磁盘一般表述为“Virtual Drive”、“Virtual Disk”（简称“VD”）、 “Volume”、“Array”、“Logical Device”（简称“LD”）。raid卡就是一个小型电脑，用作在磁盘组上创建虚拟磁盘，以及提供一些数据安全性能和缓存性能。

#### 容错性能

“鸡蛋不能放到一个篮子里”，数据安全的重要途经就是配置数据副本，最直观的影响就是数据副本会在原有磁盘空间下空间利用率降低。raid卡提供了数据副本的功能，raid卡在寻找数据安全和空间利用率之间的平衡点的过程中提供了多种冗余方式：RAID 0、1、5、6、10、50、60

#### 热备盘

是磁盘系统中的一块独立盘，raid系统中存在磁盘故障的时候自动顶上。

#### 数据重构

对于由冗余功能的raid类型(1、5、6、10、50、60)，盘坏了的时候可以把数据填充到新的硬盘或者热备盘上

#### 磁盘状态

通过命令查看物理磁盘`megacli -PDList -aALL`返回的字段中`Firmware state`有多个状态：

- Online 为某个虚拟磁盘的成员盘，可正常使用，处于在线状态。 
- Unconfigured Good 磁盘状态正常，但不是虚拟磁盘的成员盘或热备盘。 
- Hot Spare 被设置为热备盘。 
- Failed 当“Online”状态或“Hot Spare”状态的磁盘出现不可恢复的 错误时，会体现为此状态。
- Rebuild 硬盘正在进行数据重建，以保证虚拟磁盘的数据冗余性和完 整性。 
- Unconfigured Bad “Unconfigured Good”状态磁盘或未初始化的磁盘，出现无 法恢复的错误时，会体现为此状态。 
- Missing “Online”状态的磁盘被拔出后，体现为此状态。 
- Offline 为某个虚拟磁盘的成员盘，不可正常使用，处于离线状态。 
- Shield State 物理磁盘在做诊断操作时的临时状态。 
- Copyback 新盘正在替换故障成员盘

#### 磁盘条带化

磁盘条带化是为解决磁盘冲突出现的。什么是磁盘冲突呢？当多个进程访问同一个磁盘时，访问量大有可能造成冲突，产生IO瓶颈。为了解决这个问题大多数的磁盘系统会限制访问次 数（每秒的I/O操作）和数据传输率（每秒传输的数据量）。另外的一个方法就是磁盘条带化，它可以将一块连续的数据进行切割后分别放到不同的物理磁盘上，从而解决磁盘冲突，还可以增加IO并行能力。

#### 硬盘直通(JBOD)

磁盘直通又称指令透传，就是插在raid卡上的物理磁盘可以让操作系统直接管理，没有中间虚拟磁盘的中介


### 查看是否是MegaRAID(LSI)卡

如果下面两个工具不适用你的芯片代码请考虑工具`sas2ircu`或者`sas3ircu`或者`hpssacli`或者`hpacucli`


```bash
dmesg | grep RAID
[    1.669047] scsi host0: Avago SAS based MegaRAID driver

#或者，得出芯片代码
lspci |grep LSI
02:00.0 RAID bus controller: LSI Logic / Symbios Logic MegaRAID SAS 2108 [Liberator] (rev 05)
```
### 安装查看工具

我们安装LSI官方工具

### MegaCli

```bash
wget http://mirror.hep.wisc.edu/stable/extras/6/x86_64/Lib_Utils-1.00-09.noarch.rpm
wget http://mirror.hep.wisc.edu/stable/extras/6/x86_64/MegaCli-8.02.21-1.noarch.rpm

rpm -ivh Lib_Utils-1.00-09.noarch.rpm
rpm -ivh MegaCli-8.02.21-1.noarch.rpm

#64位操作系统
ln -sf /opt/MegaRAID/MegaCli/MegaCli64 /usr/sbin/megacli
```

#### 查看所有硬盘

重点关注**Predictive Failure**不为0就要准备更换硬盘

Firmware state中Online为正常状态

```bash
megacli -PDList -aALL | egrep 'Adapter|Enclosure Device ID|Coerced Size|Drive Temperature|PD Type|Slot|Inquiry|Predictive Failure|Firmware state'

#第一块raid卡的意思
Adapter #0 
#这个参数很重要，设置raid要用到
Enclosure Device ID: 32
#服务器上的硬盘位
Slot Number: 0
#这块硬盘快坏了
Predictive Failure: 2
#硬盘接口类型
PD Type: SAS
#磁盘容量
Non Coerced Size: 558.411 GB [0x45cd2fb0 Sectors]
Coerced Size: 558.375 GB [0x45cc0000 Sectors]
#磁盘厂商
Inquiry Data: SEAGATE ST3600057SS     ES656SL46W4V
Drive Temperature :47C (116.60 F)
```

#### 查看单个硬盘的信息

- 252就是`Enclosure Device ID`
- 3就是`Slot Number`

```bash
megacli -pdInfo -PhysDrv[32:0] -aALL
```

#### 查看所有虚拟磁盘

```bash
megacli -LDInfo -LALL -aAll
```

#### 查看所有磁盘raid情况

```bash
megacli -LdPdInfo -aAll -NoLog|grep -Ei '(^Virtual Disk|^RAID Level|^PD type|^Raw Size|^Enclosure|^Slot|error|firmware)' | awk '{if($0~/^Virtual/||$0~/^RAID/){printf("\033[35m%s\033[0m\n",$0)}else if($0 ~ /^Enclosure/){printf("\033[31m%s: %s\033[0m ",$1,$4)}else if($0 ~ /^Slot/){printf("\033[31m%s\033[0m\n",$0)}else if($0~/^Other/||$0~/Firmware/){printf("\033[33m%s\033[0m\n",$0)}else if($0~/^Raw/){printf("\033[33m%s%s\033[0m\n",$2,$3)}else{printf("\033[33m%s\033[0m ",$0)}}'
```

#### 定位硬盘

磁盘闪灯和关灯
```bash
#闪灯
megacli -PdLocate -start -physdrv[32:0] -a0
#关灯
megacli -PdLocate -start -physdrv[32:0] -a0
```

### storcli

storcli已经基本代替了megacli，由于加载驱动原因可能无法使用，然后使用megacli

#### 下载

到此去搜索下载最新版

```bash
https://www.broadcom.com/support/download-search
```

![storcli](https://qiniu.li-rui.top/storcli.png)

#### Linux安装

```bash
rpm -ivh  storcli-007.0709.0000.0000-1.noarch.rpm
```

#### 常用命令

```bash
storcli -v    显示软件版本信息
storcli -h    查看帮助信息
storcli show    查看RAID卡、系统内核、主机名等信息
storcli /c0 show all    查看第一块RAID卡版本、功能、状态、以及raid卡下的物理磁、逻辑盘信息。c0代表第一块raid卡，如果有多块则命令以此类推。
storcli /c0 show freespace    查看第一块RAID卡剩下的磁盘空间
storcli /c0 show rebuildrate    查看第一块RAID卡rebuildrate速度
storcli /c0 download file=3108.bin    升级第一块RAID卡固件
storcli /c0 flushcache     清除第一块RAID卡缓存
storcli /c0 /eall /sall show all     查看第一块RAID卡上物理磁盘详细信息
storcli /c0 /e252 /s0 start locate 定位第一块RAID上某块物理磁盘，物理磁盘的绿色的定位灯会闪烁。 e代表Enclosure，s代表Slot或PD
storcli /c0  /ex /sx stop locate    停止定位，定位灯停止闪烁。
storcli /c0 /e252 /sall show rebuild  查看磁盘重建进度
storcli /c0 /ex /sx start rebuild    开始重建
storcli /c0 /ex /sx stop rebuild    停止重建
storcli /c0 /ex /sx add hostsparedrive dgs=0    设置某块物理磁盘为磁盘组0的热备盘，如果不指定dgs，则为该RAID卡上全局热备盘。
storcli /c0 /ex /sx delete hostsparedrive    删除热备磁盘
storcli /c0 add vd each type=raid0 drives=252:0,1,2,3     单独为每一块物理磁盘创建raid0
storcli /c0 add vd type=raid5 size=all names=tmp1 drives=32:2-4    由第3、4、5块物理磁盘来构建RAID5，分配所有空间的逻辑磁盘命名tmp1。
storcli /c0 add vd type=raid10 size=all names=tmp1 drives=32:0-3 pdperarray=2    由前四块物理磁盘构建raid10，分配所有空间的逻辑磁盘命名为tmp1。（注意：LSI SAS3108最多支持64个RAID，创建10/50/60时，必须指定pdperarray参数。如果没有这个参数，是创建不成功的。这个参数的含义是：Specifies the number of physical drives per array. The default value is automatically chosen。）
storcli /c0 add vd type=raid10 size=100GB,200GB names=tmp1,tmp2 drives=32:0-3 pdperarray=2    由前四块物理磁盘构建raid10，分别分配多个逻辑磁盘。
storcli /c0 add vd type=raid10 size=all names=tmp3 drives=32:0-3 pdperarray=2    剩下的所有空间分配给逻辑磁盘tmp3。
storcli /c0 /vall show all     显示第一块RAID卡上所有逻辑磁盘相信信息，也可指定某个逻辑磁盘v0，v1等等。
storcli /c0 /v0 del force   强制删除某个逻辑磁盘
storcli /c0 /bbu show all   显示bbu信息
storcli /c0 /vall set wrcache=wt/wb/awb 设置写策略
storcli /c0 show alarm    查看报警器信息
storcli /c0 set alarm=silence   暂时关闭报警器鸣叫
storcli /c0 set alarm=off       始终关闭报警器鸣叫
storcli /c0 /e252 /s3 set good    改变插入的物理磁盘的状态
storcli /c0 /e252 /s3 start initialization    初始化某个物理磁盘
storcli /c0 /e252 /s3 show initialization    查看某个初始化的物理磁盘进度
storcli /c0 /v0 set wrcache=wt   修改vd的写策略
storcli /c0 /v0 set rdcache=nora   修改vd的读策略
storcli /c0 /fall show   查看foreign信息
storcli /c0 /fall import    导入foreign
storcli /c0 show termlog type=contents       在线查看日志
storcli /c0 show termlog type=contents | grep "rebuild"    在线查看日志抽取关键字
storcli /c0 show events file=/home/eventreports    将日志存储为文件
```

## 盘符和硬盘插槽对应

### smartmontools安装

>smartmontools通过使用自我监控(Self-Monitoring)、分析(Analysis)和报告(Reporting)三种技术（缩写为S.M.A.R.T或SMART）来管理和监控存储硬件。如今大部分的ATA/SATA、SCSI/SAS和固态硬盘都搭载内置的SMART系统。SMART的目的是监控硬盘的可靠性、预测磁盘故障和执行各种类型的磁盘自检。smartmontools由smartctl和smartd两部分工具程序组成，它们一起为Linux平台提供对磁盘退化和故障的高级警告。[来源](https://linux.cn/article-4461-1.html)

安装

```bash
yum install -y smartmontools
```
查看是否支持smat

```bash
#megaraid,0第一块raid卡，对于megaraid驱动需要加上megaraid,0
smartctl -i /dev/sdb -d megaraid,0

SMART support is:     Available - device has SMART capability.
SMART support is:     Enabled
```

不支持的话就手动开启

```bash
smartctl --smart=on --offlineauto=on --saveauto=on /dev/sdb -d megaraid,0
#或者
smartctl -s on /dev/sdb -d megaraid,0

#禁用
smartctl -s off /dev/sdb -d megaraid,0
```

### 基本使用

#### 健康检查

简单信息返回

```bash
smartctl -H /dev/sdb -d megaraid,0

=== START OF READ SMART DATA SECTION ===
#说明健康
SMART Health Status: OK
```

详细信息查看

```bash
smartctl -a /dev/sdb -d megaraid,0
```

#### 磁盘检测

```bash
#后台检测硬盘，消耗时间短
smartctl -t short <device> 
#后台检测硬盘，消耗时间长
smartctl -t long <device> 
#前台检测硬盘，消耗时间短
smartctl -C -t short <device> 
#前台检测硬盘，消耗时间长
smartctl -C -t long <device> 
```

### 由盘符对应硬盘插槽

#### 获取硬盘盘符

如果是ceph的osd，假设我们要获取osd.6的盘符

```bash
df /var/lib/ceph/osd/ceph-6

Filesystem     1K-blocks  Used Available Use% Mounted on
/dev/sdc1          98944  5492     93452   6% /var/lib/ceph/osd/ceph-6
```

#### 得到硬盘序列号

```bash
smartctl -i  /dev/sdc1 -d megaraid,0 | grep "Serial number"

Serial number:        6SL46W4V
```

#### 由序列号得到Enclosure Device ID 和 Slot Number

```bash
megacli -PDList -aALL|grep -B30 6SL46W4V | egrep 'Enclosure Device ID|Slot Number'

Enclosure Device ID: 32
Slot Number: 0
```

#### 硬盘开启闪灯

```bash
megacli -PdLocate -start -physdrv[32:0] -a0
```

#### python脚本

```bash
https://github.com/sunnoy/blog-issue/blob/master/get_disk_slot.py

-- Controller information --
-- ID | H/W Model            | RAM    | Temp | BBU    | Firmware
c0    | PERC H700 Integrated | 1024MB | N/A  | Good   | FW: 12.22.4-0001

-- Array information --
-- ID | Type   |    Size |  Strpsz |   Flags | DskCache |   Status |  OS Path | CacheCade        |InProgress
c0u0  | RAID-1 |    558G |   64 KB | ADRA,WB |  Default |  Optimal | /dev/sda | Type : Read Only |None
c0u2  | RAID-0 |    558G |   64 KB | ADRA,WB |  Default |  Optimal | /dev/sdb | Type : Read Only |None
c0u3  | RAID-0 |    558G |   64 KB | ADRA,WB |  Default |  Optimal | /dev/sdc | Type : Read Only |None
c0u4  | RAID-0 |    558G |   64 KB | ADRA,WB |  Default |  Optimal | /dev/sdd | Type : Read Only |None
c0u5  | RAID-0 |    558G |   64 KB | ADRA,WB |  Default |  Optimal | /dev/sde | Type : Read Only |None
c0u6  | RAID-0 |    558G |   64 KB | ADRA,WB |  Default |  Optimal | /dev/sdf | Type : Read Only |None
c0u7  | RAID-0 |    558G |   64 KB | ADRA,WB |  Default |  Optimal | /dev/sdg | Type : Read Only |None
c0u8  | RAID-0 |    558G |   64 KB | ADRA,WB |  Default |  Optimal | /dev/sdh | Type : Read Only |None
c0u9  | RAID-0 |    558G |   64 KB | ADRA,WB |  Default |  Optimal | /dev/sdi | Type : Read Only |None
c0u10 | RAID-0 |    558G |   64 KB | ADRA,WB |  Default |  Optimal | /dev/sdj | Type : Read Only |None

-- Disk information --
-- ID   | Type | Drive Model                          | Size     | Status          | Speed    | Temp | Slot ID  | LSI ID
c0u0p0  | HDD  | SEAGATE ST3600057SS ES656SL46W4V     | 558.3 Gb | Online, Spun Up | 6.0Gb/s  | 47C  | [32:0]   | 0
c0u0p1  | HDD  | SEAGATE ST3600057SS ES656SL46J2A     | 558.3 Gb | Online, Spun Up | 6.0Gb/s  | 44C  | [32:1]   | 1
c0u2p0  | HDD  | SEAGATE ST3600057SS ES656SL44ZG4     | 558.3 Gb | Online, Spun Up | 6.0Gb/s  | 48C  | [32:3]   | 3
c0u3p0  | HDD  | SEAGATE ST3600057SS ES656SL45523     | 558.3 Gb | Online, Spun Up | 6.0Gb/s  | 49C  | [32:4]   | 4
c0u4p0  | HDD  | SEAGATE ST3600057SS ES656SL46KGH     | 558.3 Gb | Online, Spun Up | 6.0Gb/s  | 47C  | [32:5]   | 5
c0u5p0  | HDD  | SEAGATE ST3600057SS ES656SL514Y1     | 558.3 Gb | Online, Spun Up | 6.0Gb/s  | 47C  | [32:6]   | 6
c0u6p0  | HDD  | HITACHI HUS156060VLS600 E516JWWEVNBJ | 558.3 Gb | Online, Spun Up | 6.0Gb/s  | 50C  | [32:7]   | 7
c0u7p0  | HDD  | SEAGATE ST3600057SS ES656SL44CXN     | 558.3 Gb | Online, Spun Up | 6.0Gb/s  | 50C  | [32:8]   | 8
c0u8p0  | HDD  | SEAGATE ST3600057SS ES656SL44PB5     | 558.3 Gb | Online, Spun Up | 6.0Gb/s  | 44C  | [32:9]   | 9
c0u9p0  | HDD  | SEAGATE ST3600057SS ES666SL94RBL     | 558.3 Gb | Online, Spun Up | 6.0Gb/s  | 48C  | [32:10]  | 10
c0u10p0 | HDD  | SEAGATE ST3600057SS ES656SL4C98J     | 558.3 Gb | Online, Spun Up | 6.0Gb/s  | 49C  | [32:11]  | 11

```
