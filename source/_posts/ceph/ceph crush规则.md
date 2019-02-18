---
title: ceph crush规则
date: 2018-11-04 12:12:28
tags:
- ceph
---
## ceph crush规则

### 提取

```bash
#1、提取已有的CRUSH map ，使用-o参数，ceph将输出一个经过编译的CRUSH map 到您指定的文件
    ceph osd getcrushmap -o crushmap.txt
#2、反编译你的CRUSH map ，使用-d参数将反编译CRUSH map 到通过-o 指定的文件中
    crushtool -d crushmap.txt -o crushmap-decompile
#3、使用编辑器编辑CRUSH map
    vi crushmap-decompile
#4、重新编译这个新的CRUSH map
    crushtool -c crushmap-decompile -o crushmap-compiled
#5、将新的CRUSH map 应用到ceph 集群中
    ceph osd setcrushmap -i crushmap-compiled
```
<!--more-->
### 规则编辑

```bash
# begin crush map
tunable choose_local_tries 0
tunable choose_local_fallback_tries 0
tunable choose_total_tries 50
tunable chooseleaf_descend_once 1
tunable chooseleaf_vary_r 1
tunable chooseleaf_stable 1
tunable straw_calc_version 1
tunable allowed_bucket_algs 54

# devices
device 0 osd.0 class hdd
device 1 osd.1 class hdd
device 2 osd.2 class hdd
device 3 osd.3 class hdd
device 4 osd.4 class hdd
device 5 osd.5 class hdd
device 6 osd.6 class hdd
device 7 osd.7 class hdd
device 8 osd.8 class hdd
device 9 osd.9 class hdd
device 10 osd.10 class hdd
device 11 osd.11 class hdd

# types
type 0 osd
type 1 host
type 2 chassis
type 3 rack
type 4 row
type 5 pdu
type 6 pod
type 7 room
type 8 datacenter
type 9 region
type 10 root

# buckets
host matr01 {
	id -3		# do not change unnecessarily
	id -4 class hdd		# do not change unnecessarily
	# weight 2.181
	alg straw2
	hash 0	# rjenkins1
	item osd.0 weight 0.545
	item osd.3 weight 0.545
	item osd.6 weight 0.545
	item osd.7 weight 0.545
}
host matr02 {
	id -5		# do not change unnecessarily
	id -6 class hdd		# do not change unnecessarily
	# weight 2.181
	alg straw2
	hash 0	# rjenkins1
	item osd.1 weight 0.545
	item osd.4 weight 0.545
	item osd.8 weight 0.545
	item osd.9 weight 0.545
}
host matr03 {
	id -7		# do not change unnecessarily
	id -8 class hdd		# do not change unnecessarily
	# weight 2.181
	alg straw2
	hash 0	# rjenkins1
	item osd.2 weight 0.545
	item osd.5 weight 0.545
	item osd.10 weight 0.545
	item osd.11 weight 0.545
}

rack rack1 {
        id -9
        alg straw
        hash 0
        item matr01  weight 2.00
    }

rack rack2 {
        id -10
        alg straw
        hash 0
        item matr02  weight 2.00
    }
rack rack3 {
        id -11
        alg straw
        hash 0
        item matr03  weight 2.00
    }

root default {
	id -1		# do not change unnecessarily
	id -2 class hdd		# do not change unnecessarily
	# weight 6.542
	alg straw2
	hash 0	# rjenkins1
	item rack1 weight 2.181
	item rack2 weight 2.181
	item rack3 weight 2.181
}

# rules
rule replicated_rule {
	id 0
	type replicated
	min_size 1
	max_size 10
	step take default
	step chooseleaf firstn 0 type rack
	step emit
}

# end crush map

```

### cursh规则命令编辑

增加3个rack，然后把host放到rack上面

```bash
#新建rack
ceph osd crush add-bucket rack01 rack
ceph osd crush add-bucket rack02 rack
ceph osd crush add-bucket rack02 rack

#将host放到rack下面
ceph osd crush move ceph-node1 rack=rack01
ceph osd crush move ceph-node2 rack=rack02
ceph osd crush move ceph-node3 rack=rack03

#将rack放到root节点下面

ceph osd crush move rack03 root=default
ceph osd crush move rack02 root=default
ceph osd crush move rack01 root=default
```

配置的含义就是选择副本数量的节点类型，如果上面加入了rack那么必须保证为至少3个rack

```bash
osd crush chooseleaf type: 3
```

cursh规则

```bash
rule ssd {  
	#规则代号  
    ruleset 0   
	#类型为副本模式，另外一种模式为纠删码EC   
    type replicated 
	#如果一个归置组副本数小于此数， CRUSH 将不应用此规则。    
    min_size 1  
	#如果一个归置组副本数大于此数， CRUSH 将不应用此规则。  
    max_size 10 
	#选取桶名为入口并迭代到树底。   
    step take ssd
	#选取指定类型桶的数量，这个数字通常是存储池的副本数（即pool size ）。选择{bucket-type} 类型的一堆桶，并从各桶的子树里选择一个叶子节点。集合内桶的数量通常是存储池的副本数（即pool size ）。   
    step chooseleaf firstn 0 type rack 
	#输出当前值并清空堆栈。通常用于规则末尾，也适用于相同规则应用到不同树的情况。   
	step emit  
}

```



