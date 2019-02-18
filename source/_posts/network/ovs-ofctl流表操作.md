---
title: ovs-ofctl流表操作
date: 2018-12-13 12:14:28
tags:
- openflow
---

# ovs-ofctl使用

ovs使用分为两类

- 通过命令来直接操作ovs
- 使用控制器通过openflow协议或其他协议去远程操作ovs

<!--more-->

# 控制器操作

到目前为面世的控制器有  

- [ovn](http://www.openvswitch.org//support/slides/OVN-Vancouver.pdf) ovs出品
- [ONOS](https://onosproject.org/)
- [OpenDaylight](https://www.opendaylight.org/)
- [Floodlight](http://www.projectfloodlight.org/floodlight/)
- [Ryu](https://osrg.github.io/ryu/)

# ovs-ofctl命令

[ovs命令文档](http://www.openvswitch.org/support/dist-docs/)

![ovs](https://qiniu.li-rui.top/ovszjjg.png)

该命令用来配置和维护OpenFlow交换机，由于ovs-ofctl使用openflow协议，因此可以对任何openflow交换机进行配置，不仅仅是openvswitch
下面根据[官方文档](http://www.openvswitch.org/support/dist-docs/ovs-ofctl.8.html)整理

# 交换机管理

```bash
#查看交换机
#frags=normal配置对待分片模式
ovs-ofctl show br-lan

#查看交换机流表
ovs-ofctl dump-tables br-lan

#查看流表特性。openflow1.3+
ovs-ofctl dump-table-features br-lan

#查看流表详细信息 openflow1.4+
ovs-ofctl dump-table-desc br-lan

#更改流表设置setting可取值drop continue controller等
ovs-ofctl mod−table switch table_id setting

#查看交换机端口
ovs-ofctl dump-ports br-lan

#查看端口详细信息
ovs-ofctl dump-ports-desc br-lan

#更改交换机端口动作
#action可取值 up/down stp/on-stp receive forward flood packet−in
ovs-ofctl mod−port switch port action

#配置交换机对待分片模式
#分片模式可选 
#normal 分片的包和普通报一样对待
#drop 分片的包就会被丢弃
#reassemble 分片组装完再进入流表，ovs不提供这个模式
ovs-ofctl set−frags switch fra g_mode
#获取分片模式
ovs-ofctl get−frags switch

#查看流表
#[−−sort and −−rsort] 顺序打出来
ovs-ofctl dump-flows br-lan [−−sort and −−rsort]
ovs-ofctl dump-flows br-lan table=0 [−−sort and −−rsort]

#查看队列
queue−stats switch [port [queue]]
ovs-ofctl queue-stats br-lan
```

# 流表

## 流表命令

### 流表操作主要命令有

- add−flow switch flow 
- mod−flows switch flow
- del−flows switch flow
- replace−flows switch flow
- diff−flows switch flow

### 流表查看命令

- ovs-ofctl dump-flows

### 流表添加语法

流表中流的格式主要是`字段=值`格式，中间使用空格或逗号隔开

```bash
#dong 是交换机
ovs-ofctl add-flow dong priority=12,in_port=2,actions=drop
```

字段匹配通配符规则：只有底层字段有确定值时，高层字段才可以有通配符

## 流表中的字段

流表中的字段有下面的属性
- 名称
- 字段大小 一般是协议头中的字段大小
- 格式 十进制 十六进制 Ethernet ipv4 ipv6等
- 通配符的使用
- 必备条件 决定字段值的格式 一些字段的值是在固定的几个里面选择的
- 访问权限 可读/刻写该字段值
- OXM(OpenFlow Extensible Match)/NXM(Nicira Extended Match) 字段名称 

一个字段可能有三个名称 普通名称、OXM名称、NXM名称。Nicira是ovs的公司，现在被VMware收购。比如字段隧道id的名称：
- name: tun_id (别名 tunnel_id)
- OXM: OXM_OF_TUNNEL_ID
- NXM: NXM_NX_TUN_ID

### 基本字段

下面字段在流表输出显示
- duration=156316.793s 条目在流表中的时间，精确到纳秒(10的负9次方秒)
- n_packets=2 匹配该条目的网络包数量
- n_bytes=84 匹配该条目的数据量
- idle_age=65534 该flow没有匹配网络包的时间，秒数
- hard_age=65534 该flow新加入或修改后到当前的时间，秒数
下面字段为添加流表字段
- table 指定流表，0到254
- cookie 用来给一些列的flow来添加标记， 默认为0 
- priority 包匹配flow优先级，值越大越高，取值范围0-65535，默认32786
- idle_timeout 改flow没有匹配到过期的时间，0永不过期
- hard_timeout 无论是否有匹配到给定时间就过期，0永不过期
- importance 交换机对流表中flow的驱逐机制(eviction mechanism)中的因子

流表中的flow删除有三种方法：控制器发请求删除(OFPFC_DELETE or OFPFC_DELETE_STRICT)；通过交换机中的过期机制( idle_timeout and a hard_timeout)；通过交换机中的flow的驱逐机制。importance字段中就是该机制的因子，当交换机需要进行资源回收时就会发生该动作。importance字段值为0为不会被驱逐

- reset_counts 重置flow匹配计数
- out_port 该flow必须要有out port 动作，port为openflow端口

### 匹配字段

匹配字段类型可以从以太网到arp再到ip和tcp，大多来自于协议包的头部，常用的匹配字段有

#### SFC中nsf字段

什么是sfc(Service Function Chain)?网络服务中既包含网络防火墙、nat等功能，也包含网络应用服务比如负载均衡等。流量一次通过这些功能和服务，这就是服务链(Service Function Chain)。

![sfc](https://qiniu.li-rui.top/sfc.png)

在网络包遍历整个Chain时，携带的以供Service Function使用的的信息。 SFC Header可以是基于现有的网络协议中的Header，例如IPv4 header中的DSCP bits，或者是MPLS labels。

- nsh_flags nsh标志
- nsh_mdtype
- nsh_np 下一个协议(next protocol)
- nsh_spi (aka nsp) 服务路径标识(service path identifier)
- nsh_si (aka nsi) 服务索引(service index)
- nsh_c1 (aka nshc1) Network Platform Context
- nsh_c2 (aka nshc2) Network Shared Context
- nsh_c3 (aka nshc3) Service Platform Context
- nsh_c4 (aka nshc4) Service Shared Context

#### 隧道字段

隧道就是一层协议作为同层或者其他层的链路层，比如在IP里面传递二层数据，隧道是可以多层使用的，常用的隧道有gre和vxlan。在处理隧道和openflow端口映射上，ovs支持一些列的使用模式

- Port-based 一个openflow端口表示一个隧道
- Flow-based 一个openflow端口表示所有可能的隧道
- Mixed  Port-based和Flow-based可以并存
- Intermediate 一个opeflow部分配置为Flow-based 

更多隧道配置[相见](http://www.openvswitch.org/support/dist-docs/ovs-vswitchd.conf.db.5.html)

- tun_id (aka tunnel_id)，gre中是key，vxlan中是vni
- tun_src 隧道载体v4源地址
- tun_dst 隧道载体v4目的地址
- tun_ipv6_src
- tun_ipv6_dst
- tun_gbp_id vxlan中bgp id
- tun_gbp_flags vxlan中gbp 标志

vxlan gbp(Group-Based Policy )，使用vxlan报文中的保留字段，下面为对比

![vxlan](https://qiniu.li-rui.top/vxlan.png)

![vxlangbp](https://qiniu.li-rui.top/vxlangbp.png)

- tun_metadata0 Geneve隧道的字段

VxLAN、NvGRE、STT、Geneve隧道对比分析[详见](https://www.sdnlab.com/15820.html)

#### 元数据字段

元数据字段不是从数据报文中提取的，而是与数据包的来源和处理有关

- in_port Ingress端口
- in_port_oxm openflow中Ingress端口
- skb_priority Output Queue 出口队列
- pkt_mark 数据包标记
- actset_output action设置出口
- packet_type 数据包类型

#### 连接追踪字段

从ovs2.5开始加入了该字段，openflow里面没有

- ct_state 连接状态，可选new est rel rpl inv trk snat dnat
- ct_zone 连接域
- ct_mark
- ct_label
- ct_nw_src 追踪原始地址IPv4源地址
- ct_nw_dst
- ct_ipv6_src
- ct_ipv6_dst
- ct_nw_proto 原始IP协议，6 为tcp
- ct_tp_src 源端口
- ct_tp_dst 目的端口

#### 寄存器字段

openflow中使用寄存器register来在流表pipeline之间传递信息

- metadata openflow中比较老的寄存器字段
- reg0 第一个寄存器字段
- xreg0 寄存器拓展字段，openflow1.5引入名称为packet registers，和ovs内现有字段重复，因此命令为拓展字段
- xxreg0 双拓展字段128bits

#### 以太网字段

- eth_src (aka dl_src) 包源mac
- eth_dst (aka dl_dst) 包目的mac
- eth_type 以太网类型，该字段支持简写

```bash
#左边是简写
eth packet_type=(0,0) (Open vSwitch 2.8 and
ip eth_type=0x0800
ipv6 eth_type=0x86dd
icmp eth_type=0x0800,ip_proto=1
icmp6 eth_type=0x86dd,ip_proto=58
tcp eth_type=0x0800,ip_proto=6
tcp6 eth_type=0x86dd,ip_proto=6
udp eth_type=0x0800,ip_proto=17
udp6 eth_type=0x86dd,ip_proto=17
sctp eth_type=0x0800,ip_proto=132
sctp6 eth_type=0x86dd,ip_proto=132
arp eth_type=0x0806
rarp eth_type=0x8035
mpls eth_type=0x8847
mplsm eth_type=0x8848
```

#### vlan字段

- vlan_vid
- vlan_pcp vlan优先级openflow1.2引入

#### MPLS字段

什么是MPLS？Multiprotocol Label Switching多协议标签交换，是一种路由协议。传统路由使用最长前缀匹配(LPM，Longest prefix match)算法来找到下一跳的地址或接口，这个算法使用软件实现，量大就会效率低，而且要维护很多路由表条目。mpls工作在2.5层，是一种使用标签来路由数据包而不是使用IP地址的协议。

- mpls_label 标签
- mpls_tc Traffic Class
- mpls_bos Bottom of Stack
- mpls_ttl mpls中的ttl

#### ip字段

- ip_src (aka nw_src) 包源地址
- ip_dst (aka nw_dst) 包目的地址
- ipv6_src
- ipv6_dst
- ipv6_label v6标签字段
- nw_proto (aka ip_proto) IP协议字段
- nw_ttl IP中ttl和hop限制
- ip_frag IP包分片匹配 no 仅匹配完整包，  yes 匹配所有分片 ，first 匹配分片offset为0 ，later 非零offset ，not_later offset为0和不分片
- nw_tos IP中dcsp 2-7 bits
- ip_dscp IP中dscp 0-5 bits

dscp(Differentiated Services Code Point)为服务差分，主要是对不同的流量进行分类从而设定优先级，详细的qos[请参考](https://cshihong.github.io/2018/02/05/QoS%E6%A6%82%E8%BF%B0/)

- nw_ecn (aka ip_ecn) IP中ecn

什么是ecn？Explicit Congestion Notification，是通过TCP发送端和接收端以及中间路由器的配合，感知中间路径的拥塞，并主动的减缓TCP的发送速率，从而从早期避免拥塞而导致的丢包

#### arp字段

- arp_op arp操作类型 1 请求，2 应答，3 rarp请求，4 rarp应答
- arp_spa arp源地址
- arp_tpa arp目的地址
- arp_sha arp源mac
- arp_tha arp目的mac

#### 4层字段

- tcp_src (aka tp_src) tcp源端口
- tcp_dst (aka tp_dst) tcp目的端口
- tcp_flags tcp标志
- udp_src udp源端口
- udp_dst udp目的端口
- sctp_src sctp源端口
- sctp_dst sctp目的端口

sctp(Stream Control Transmission Protocol，流控制传输协议)，具有tcp和udp的特性，可认为是tcp的改进，但又和tcp有很大区别。相对于tcp sctp主要有多宿主、四次握手、多流等特点。

#### icmp字段

- icmp_type icpm报文type
- icmp_code icpm报文code

icmp中type和code[请在此查看](https://github.com/sunnoy/books_note_res/blob/master/icmp-parameters.txt)

- icmpv6_type
- icmpv6_code 

icmpv6中type和code[请在此查看](https://github.com/sunnoy/books_note_res/blob/master/icmpv6-parameters.txt)

- nd_target icmpv6邻居发现(Neighbor Discovery)目的
- nd_sll icmpv6邻居发现源地址
- nd_tll icmpv6邻居发现目的地址

匹配字段基本就是这么多，个别字段的使用对openflow和ovs的版本有要求，字段的详细参考[请查看这个pdf](https://github.com/sunnoy/books_note_res/blob/master/ovs-fields.7%E6%9C%89%E7%9B%AE%E5%BD%95.pdf)，该pdf相对官网上的版本，笔者在上面加入了目录。

![labler](https://qiniu.li-rui.top/labler.png)

### 动作字段

对一个flow匹配后的动作字段可以有一系列的动作行为。如果没有出现action，那么这个flow的包就会被drop掉。

- output:port 数据包转发到指定的openflow端口
- output(port=port,max_len=nbytes) 从port=里面读取端口，如果数据包大于nbytes则进行裁剪到nbytes
- group:group_id 将包转发到group id
- normal 控制数据包进行正常的2/3层过程
- flood 输出数据包到所有的端口，stp开启的端口会收不到这些包
- all 输出数据包到所有端口
- local 数据包输出到与网桥同名的端口
- in_port 将数据包返回给输入口
- controller(max_len=nbytes) 向控制器发送信息,max_len=nbytes 等
- enqueue(port,queue) 将数据包放入交换机的队列中
- drop 丢弃包
- mod_vlan_vid:vlan_vid 更改vlan id
- mod_vlan_pcp:vlan_pcp 更改vlan优先级
- strip_vlan 除去vlan信息
- push_vlan:ethertype 打上vlan标签，是0x8100 新的vlan 优先级和vni都是0
- push_mpls:ethertype 将数据包的Ethertype改成<ethertype>，只能是0x8847或者0x8848，同时添加MPLS LSE。
- pop_mpls:ethertype 除去mpls标签
- mod_dl_src:mac 改源mac
- mod_dl_dst:mac 改目的mac
- mod_nw_src:ip 改源IP
- mod_nw_dst:ip 改目的ip
- mod_tp_src:port 改源TCP、UDP、SCTP端口
- mod_tp_dst:port 改目的TCP、UDP、SCTP端口
- mod_nw_tos:tos 设置tos 4的倍数在0-255之间
- mod_nw_ecn:ecn 设定ecp，0-3
- mod_nw_ttl:ttl 设定IP v4/v6的ttl
- resubmit:port
- resubmit([port],[table])
- resubmit([port],[table],ct) 替换in_port为port将数据包定向到下一个流表或者端口
- set_tunnel:id 设定隧道
- set_tunnel64:id 设定64bits隧道
- set_queue:queue 设定队列
- pop_queue 恢复set_queue的队列
- ct([argument][,argument...]) 将包加入追踪器，可选值有 commit force等
- ct_clear 清除追踪器
- dec_ttl 减少ttl值
- set_mpls_label:label 设置mpls标签
- set_mpls_tc:tc 设置traffic-class 0-7
- set_mpls_ttl:ttl 设置ttl 0-255
- dec_mpls_ttl 减少mpls的ttl
- set_field:value[/mask]−>dst load:value−>dst[start..end] 设定字段值

该动作有两种使用方式

```bash
#set 指令可以是文字到普通字段
set_field:00:11:22:33:44:55->eth_src

#load 指令需要传入十进制和0x开头的十六进制，字段名称要使用NXM or OXM名称
load:0x001122334455->OXM_OF_ETH_SRC[]
```

- goto_table:table 到下一个流表
- write_metadata:value[/mask] 写入flow的元数据
- learn(argument[,argument]...) 改动作类似ovs−ofctl −−strict
mod−flows

更多的动作字段[请点击查看](https://github.com/sunnoy/books_note_res/blob/master/ovs-ofctl.8%E6%9C%89%E7%9B%AE%E5%BD%95.pdf)，该pdf笔者已经加入目录




