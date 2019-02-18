---
title: 理解ceph中pg
date: 2018-11-04 12:12:28
tags:
- ceph
---
## 理解ceph中pg

### 对象

![ceph总](https://qiniu.li-rui.top/ceph总.png)

ceph所支持的存储类型中，所有通过这些不通形式导入到ceph的文件都会在客户端分割成为对象。最终放入到RADOS中。
<!--more-->
### RADOS

对象最终还是要落在系统中的磁盘上，准确来说是放在osd上面。

ceph在客户端到osd之间进行了一些抽象与封装来更好的保证数据安全和冗余。

#### pool

pool是存储空间的逻辑划分，一个集群可以分成多个pool，也可以使用单个pool，但必须要有一个pool

pool与数据安全策略相联系，定义池就要同时定义出pool的pg数量和数据冗余策略，数据冗余策略包括副本数和纠删码，以及使用的crush规则

创建pool

```bash
#ceph osd pool create {pool-name} {pg-num} [{pgp-num}] [replicated]        [crush-ruleset-name]
#replicated指定为副本数类型
#replicated_rule为cursh规则
ceph osd pool create images 128 128 replicated replicated_rule
ceph osd pool application enable images rbd
rbd pool init images

```

查看pool

```bash
#列出pool
ceph osd lspools

#查看pool详细信息
 ceph osd pool ls detail
#获取pool的属性
ceph osd pool get {pool-name} pg_num

#更改pool的属性
ceph osd pool set {pool-name} {key} {value}

#设置pool副本数
ceph osd pool set data size 3
#pool最低副本数
ceph osd pool set data min_size 2
#设置最大对象数量
ceph  osd  pool  set-quota  images  max_objects  10000

#删除pool
ceph osd pool delete {pool-name} [{pool-name} --yes-i-really-really-mean-it]
```

#### pg

pool有pg构成，对象存到pool是存到特定的pg中。可以理解为对象的虚拟目录。在创建pool的时候就要把pg数量规划好。pg数量只可以增大不可以缩小。

#### OSD

pg是pool的组成部分，但确是落在不同OSD上面的。

添加OSD

```bash
ceph-disk prepare --bluestore /dev/sdb --block.db /dev/sdb --block.wal /dev/sdc
```
查看OSD

```bash
ceph osd tree 
#详细查看
ceph osd dump
```

激活OSD

```bash
#激活100MB分区
ceph-disk activate /dev/sdb1
```

删除OSD

```bash
ceph osd out {osd number}
systemctl stop ceph-osd@{osd number}
ceph osd crush remove osd.{osd number}
ceph auth del osd.{osd number}
ceph osd rm osd.{osd number}
ceph osd crush remove {hostname}

```

重置缓存盘
```bash
#重置OSD盘数据盘
ceph-disk zap /dev/sdb

#重置缓存盘

```

#### pgp

ceph三副本情况下每一个pg都会存放到不同的OSD上去，我们把它看成一个OSD组合。比如pg.1的OSD组合为(0.3.9)。pg数量都是设置好的，如果要保证每一个pg所存的OSD组合都是不一样的，那么就需要OSD组合数和pg数量一样。这个OSD组合数就是pgp

![pgosd](https://qiniu.li-rui.top/pgosd.png)

向pool里面写入文件，该文件就会被分割成很多个对象，这些对象会根据cursh规则来算出分别属于哪个pg，然后pg也会根据cursh规则算出pg放到那些OSD上面

### pg数量计算

单个OSD所能承载的pg数量默认是200，这个可以通过配置

```bash
mon_max_pg_per_osd=300;
```
来配置

假设集群中OSD数量为9个，分成三个pool，数据副本数量为3那么：

pool可以用的总的pg = OSD数量9×200/副本数量3=600
每个pool可以存放的pg数=600/3=200

pg数量需要是2的次幂，pg数量设置为128，pgp也为128

不同的pool要根据业务需求配置不同的pg数量来保证数据在OSD中分布均衡。

ceph官方提供的计算工具

```bash
https://ceph.com/pgcalc/
```

### 统计OSD上的pg数量脚本

```bash
ceph pg dump | awk '
 /^pg_stat/ { col=1; while($col!="up") {col++}; col++ }
 /^[0-9a-f]+\.[0-9a-f]+/ { match($0,/^[0-9a-f]+/); pool=substr($0, RSTART, RLENGTH); poollist[pool]=0;
 up=$col; i=0; RSTART=0; RLENGTH=0; delete osds; while(match(up,/[0-9]+/)>0) { osds[++i]=substr(up,RSTART,RLENGTH); up = substr(up, RSTART+RLENGTH) }
 for(i in osds) {array[osds[i],pool]++; osdlist[osds[i]];}
}
END {
 printf("\n");
 slen=asorti(poollist,newpoollist);
 printf("pool :\t");for (i=1;i<=slen;i++) {printf("%s\t", newpoollist[i])}; printf("| SUM \n");
 for (i in poollist) printf("--------"); printf("----------------\n");
 slen1=asorti(osdlist,newosdlist)
 delete poollist;
 for (j=1;j<=slen;j++) {maxpoolosd[j]=0};
 for (j=1;j<=slen;j++) {for (i=1;i<=slen1;i++){if (array[newosdlist[i],newpoollist[j]] >0  ){minpoolosd[j]=array[newosdlist[i],newpoollist[j]] ;break } }};	
 for (i=1;i<=slen1;i++) { printf("osd.%i\t", newosdlist[i]); sum=0; 
 for (j=1;j<=slen;j++)  { printf("%i\t", array[newosdlist[i],newpoollist[j]]); sum+=array[newosdlist[i],newpoollist[j]]; poollist[j]+=array[newosdlist[i],newpoollist[j]];if(array[newosdlist[i],newpoollist[j]] != 0){poolhasid[j]+=1 };if(array[newosdlist[i],newpoollist[j]]>maxpoolosd[j]){maxpoolosd[j]=array[newosdlist[i],newpoollist[j]];maxosdid[j]=newosdlist[i]};if(array[newosdlist[i],newpoollist[j]] != 0){if(array[newosdlist[i],newpoollist[j]]<=minpoolosd[j]){minpoolosd[j]=array[newosdlist[i],newpoollist[j]];minosdid[j]=newosdlist[i]}}}; printf("| %i\n",sum)} for (i in poollist) printf("--------"); printf("----------------\n");
 slen2=asorti(poollist,newpoollist);
 printf("SUM :\t"); for (i=1;i<=slen;i++) printf("%s\t",poollist[i]); printf("|\n");
 printf("Osd :\t"); for (i=1;i<=slen;i++) printf("%s\t",poolhasid[i]); printf("|\n");
 printf("AVE :\t"); for (i=1;i<=slen;i++) printf("%.2f\t",poollist[i]/poolhasid[i]); printf("|\n");
 printf("Max :\t"); for (i=1;i<=slen;i++) printf("%s\t",maxpoolosd[i]); printf("|\n");
 printf("Osdid :\t"); for (i=1;i<=slen;i++) printf("osd.%s\t",maxosdid[i]); printf("|\n");
 printf("per:\t"); for (i=1;i<=slen;i++) printf("%.1f%\t",100*(maxpoolosd[i]-poollist[i]/poolhasid[i])/(poollist[i]/poolhasid[i])); printf("|\n");
 for (i=1;i<=slen2;i++) printf("--------");printf("----------------\n");
 printf("min :\t"); for (i=1;i<=slen;i++) printf("%s\t",minpoolosd[i]); printf("|\n");
 printf("osdid :\t"); for (i=1;i<=slen;i++) printf("osd.%s\t",minosdid[i]); printf("|\n");
 printf("per:\t"); for (i=1;i<=slen;i++) printf("%.1f%\t",100*(minpoolosd[i]-poollist[i]/poolhasid[i])/(poollist[i]/poolhasid[i])); printf("|\n");
}'
```

### pg的几个状态

#### Degraded

pg存在3个OSD中，有一个OSD挂了就是

#### Peered

pg存在3个OSD中，有两个OSD挂了就是

#### Remapped

pg存在3个OSD中，有一个OSD挂了，然后他又上线了，就会

#### Recover

是pg对pg内的对象进行更新

### ceph集群

#### 状态查看

```bash
ceph -s
#实时状态查看
ceph -w
#空间查看
ceph df
```




