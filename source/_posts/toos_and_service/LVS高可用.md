---
title: LVS高可用
date: 2018-11-04 12:12:28
tags:
- lvs
- keepalived
---
# LVS高可用

## 概览

LVS(Linux Virtual Server)一个开源的负载均衡项目，架构上分为调度层，server集群层，共享存储层。
LVS有两部分组成：在内核空间工作的IPVS，真正负责业务的区域，在用户空间工作的ipvsadm，负责定义流向规则与服务器属性。
<!--more-->
### 数据流向

请求流向为：用户请求->进入内核空间->PREROUTING首先接到用户请求，判断请求内的IP是本机IP->把数据发给INPUT链->ipvs和input链在一块，判断请求是否为集群服务，并且修改数据包内的目标IP地址和端口，将修改后的数据包发给POSTROUTING链->POSTROUTING链将目标IP传递给后端服务器

### 节点表征

- CIP(Client IP)访问客户端的IP地址
- VIP(虚拟IP)外部直接面向用户的IP
- DS(director server)前端负载均衡器节点
- DIP(Director Server IP)主要用于和后端主机通讯的IP地址
- RIP(Real Server IP)后端服务器的IP地址
- RS(Real Server)后端真实的工作服务器

### 三种方式

- NAT网络地址转换，分为两种nat和fullnat
- DIP修改为RIP，反向代理，请求响应均通过DS
- DR直接路由，修改MAC，源MAC地址为DIP的MAC地址，目标MAC地址为RIP的MAC地址
- TUN隧道，在报文上再加一层IP，内部IP首部(源地址为CIP，目标IIP为VIP)，外层IP首部(源地址为DIP，目标IP为RIP)。

### 十种中调度方式

- 轮叫调度 rr
- 加权轮叫 wrr
- 最少链接 lc
- 加权最少链接 wlc
- 基于局部性的最少连接调度算法 lblc
- 复杂的基于局部性最少的连接算法 lblcr
- 目标地址哈希 dh
- 源地址哈希 sh
- 最短预期延迟 sed
- 忙时不要排队 nq



### 安装

仅需安装用户空间管理工具ipvsadm

```bash
yum install -y ipvsadm
```

## 模式

### 概览

NAT模式将所有的请求都会进入负载均衡器内，并把数据包进行重写，把目标IP和改为后端服务器IP

后端服务器响应的数据包给负载均衡器，负载均衡器将改动
源IP为负载均衡器IP发送给客户端

支持端口映射，只需要在负载均衡器上配置公网IP
所有RS节点的网关指向LVS调度器，

NAT模式分为两种一种nat和fullnat模式，前一种本质上是DNAT，后一种DNAT和SNAT都有，fullnat最大的优势为后端服务器和负载均衡可以跨vlan，同时fullnat会有百分之10的性能损失相对于nat 

![fullnat](https://qiniu.li-rui.top/fullnat.png)

[fullnat阿里已经开源](https://github.com/alibaba/LVS)，具体文档[见](https://github.com/alibaba/LVS/blob/master/docs/LVS_FULLNAT.pdf)

### 环境

- DS外网IP，200
- DS内网IP，8
- RS内网IP，18
- RS内网IP，28
- **RS网关为DS内网IP，8**

### DS实现NAT

- 开启路由转发

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

### 当客户端，DS，RS在一个网段


- 删除rs中和客户端同网段的路由

```bash
#删除之前

default via 11.11.11.10 dev eth1
192.168.66.0/24 dev eth0 proto kernel scope link src 192.168.66.75
11.11.11.0/24 dev eth1 proto kernel scope link src 11.11.11.12
169.254.0.0/16 dev eth0 scope link metric 1002

#删除路由192.168.66.0/24
ip route del 192.168.66.0/24

#删除之后
default via 11.11.11.10 dev eth1
11.11.11.0/24 dev eth1 proto kernel scope link src 11.11.11.12
169.254.0.0/16 dev eth0 scope link metric 1002

```

- 关闭ICMP重定向

```bash
echo 0 > /proc/sys/net/ipv4/conf/all/send_redirects
echo 0 > /proc/sys/net/ipv4/conf/default/send_redirects
echo 0 > /proc/sys/net/ipv4/conf/eth0/send_redirects
echo 0 > /proc/sys/net/ipv4/conf/eth1/send_redirects
```



> 当客户端，DS，RS均在一个网段中需要开启，
> **为什么要关闭ICMP重定向**
> 因为防止负载均衡失效
> > 当Real-server将Reply发送回LVS时，Reply包是 RIP -> CIP的，LVS看到RIP-> CIP实际上根本没必要经过LVS，直接到网关就行了，因为大家在一个网段，所以产生ICMP重定向发送给Real-server；

>>  Real-server收到ICMP重定向包后，如果Real-server的ICMP重定向开启了，Real-server就会处理ICMP重定向包，直接将Reply包发给网关，这时Reply包头并没有被LVS重写，所以LVS负载出现了问题。

- ipvsadm配置

```bash
IPVSADM='/sbin/ipvsadm'
$IPVSADM -C
$IPVSADM -A -t 172.16.254.200:80 -s wrr
$IPVSADM -a -t 172.16.254.200:80 -r 192.168.0.18:80 -m -w 1
$IPVSADM -a -t 172.16.254.200:80 -r 192.168.0.28:80 -m -w 1

#######################
也可以这样
echo "
       -A -t 207.175.44.110:80 -s rr
       -a -t 207.175.44.110:80 -r 192.168.10.1:80 -m
       -a -t 207.175.44.110:80 -r 192.168.10.2:80 -m
       -a -t 207.175.44.110:80 -r 192.168.10.3:80 -m
       -a -t 207.175.44.110:80 -r 192.168.10.4:80 -m
       -a -t 207.175.44.110:80 -r 192.168.10.5:80 -m
       " | ipvsadm -R


```

> **ipvsadm使用**
>
> LVS集群管理
>> ipvsadm -A|E -t|u|f service-address [-s scheduler] [-p [timeout]]
> -A/D/E/C 添加/删除/修改/清空集群
> > ipvsadm -A -t 192.168.0.198:80 -s rr
> > -t为使用TCP，-u为使用UDP，+VIP地址，-s 指定调度策略
> -p 持久链接，默认为300s在指定的时间指定用
> 户访问均为同一个RS
>
> RS管理
> > ipvsadm -a|e -t|u|f service-address -r server-address [-g|i|m] [-w weight] [-x upper] [-y lower]
> ipvsadm -a -t 192.168.0.199:80 -r 192.168.0.164:80 -g
 >> -a 添加RS后端服务器
 > -t指定在哪个lvs集群上添加RS
 > -r RS的地址IP
 > -g/i/m 分别为三种转发方式，直接路由/隧道/网络地址转换

 > 查看规则
 > > ipvsadm -ln

 > 保存规则和还原规则
 > > ipvsadm -S /-R


## keepalived

- 配置文件路径

```bash
/etc/keepalived
|-------keepalived.conf #主配置文件
|-------host
|-------|---172.17.2.236_80.conf #虚拟IP
|-------|---172.17.2.236_80 #虚拟IP内的RS主机配置
|-------|---|---172.18.2.198_8080.conf
|-------|---|---172.18.2.97_8080.conf
```

- 主配置文件**keepalived.conf**

```bash
# 参数对照表
# route_id_wan，接入路由id，1-255任意整数
# route_id_lan，内网路由id，1-255任意整数
# state，状态，可选值MASTER,BACKUP
# priority，有state决定，master100，backup50
# src_ip_wan，源接入ip
# peer_ip_wan，匹配接入ip
# src_ip_lan，源内网ip
# peer_ip_lan，匹配内网ip
# virtual_ip_wan，虚拟接入ip，例如172.17.2.79
# virtual_ip_lan，虚拟内网ip，例如172.18.2.79
# virtual_maskint_wan，虚拟接入掩码位数
# virtual maskint_lan，虚拟内网掩码位数
#
global_defs {
    router_id LVS_135_51
}
#定义一个内网+外网虚拟同步组，
#当存在任意一个网段出现问题就一起切换
vrrp_syncv_group SWJ {
    group {
        WAN_GATEWAY
        LAN_GATEWAY
    }
}

#定义一个外网实例
vrrp_instance WAN_GATEWAY {
    #第一状态
	state BACKUP
	#发送VRRP报文接口
    interface eth0
	#虚拟路由id，虚拟路由组内要保持一直
    virtual_router_id 135
	#权重，越大就会排到前面
    priority 100
	#非抢占模式，即便master起来了也不抢占banckup
    nopreempt
	#发送VRRP报文的间隔，秒
    advert_int 3
	#单薄IP也就是，发送VRRP报文的IP
    unicast_src_ip 172.17.2.41
    unicast_peer {
	    #同一组的虚拟路由
        172.17.2.79
    }
	#认证方式，PASS密码认证
    authentication {
        auth_type PASS
		#认证密码
        auth_pass gainet135
    }
	#虚拟IP地址，对客户端暴露的地址
    virtual_ipaddress {
	    #后面接网卡接口，label就是附着在前面的子网卡上
        172.17.2.236/16 dev eth0 label eth0:1
    }
	#当虚拟IP飘过来时，进行的动作脚本，一个同步组内只需要定义一个
	#此为推送到业务平台进行更改显示状态，并且写入日志
    notify_master "/bin/bash /opt/lvs/scripts/change_master.sh"
	#脚本内容
	#!/bin/bash
	#curl -X PUT -H "Content-Type: application/json" -d '{"switch_lvs_role":{"current_master_extranet_ip": "172.17.2.41", "current_slave_extranet_ip": "172.17.2.79", "vip_extranet":"172.17.2.236", "vip_intranet":"172.18.2.236"}}' 'http://172.17.1.5:38085/restfulapi/v2/lvs/operate?key=pc.zzidc.com&secret=A-quick-brown-fox-jumps-over-the-lazy-dog.'

	#echo "172.17.2.41 change to master at `date '+%Y-%m-%d %H:%M:%S'`" >> /var/log/lvs.log

}
vrrp_instance LAN_GATEWAY {
    state BACKUP
    interface eth1
    virtual_router_id 51
    priority 100
    nopreempt
    advert_int 3
    unicast_src_ip 172.18.2.41
    unicast_peer {
        172.18.2.79
    }
    authentication {
        auth_type PASS
        auth_pass gainet51
    }
    virtual_ipaddress {
        172.18.2.236/16 dev eth1 label eth1:1
    }
}

#包含的配置文件

include /etc/keepalived/hosts/*.conf

```

- VIP虚拟服务配置文件**172.17.2.236_80.conf**


```bash
#指定虚拟服务器，IP+空格+端口
#就是ipvsadm -A -t 172.17.2.236:80 -s rr
virtual_server  172.17.2.236 80 {
    #服务健康检查周期，秒
	delay_loop 6
	#调度算法
    lb_algo rr
	#转发模式
    lb_kind NAT
	#网络掩码
    nat_mask 255.255.0.0
    #长连接时间
	persistence_timeout 50
    #转发协议
	protocol TCP
    #包含后端服务器
    include /etc/keepalived/hosts/172.17.2.236_80/*.conf
}

```

- RS主机配置文件**172.18.2.198_8080.conf**

```bash
#后端服务器
#就是ipvsadm -a -t 172.17.2.236:80 -r 172.18.2.198:8080 -w 1
real_server 172.18.2.198 8080 {
    #后端服务器健康检查  成功  时的脚本，后面接脚本中定义的五个变量赋值
	notify_up "/bin/bash /opt/lvs/scripts/rs_state_change.sh 172.17.2.236 172.18.2.198 1 80 8080"
    #后端服务器健康检查  失败  时的脚本
	notify_down "/bin/bash /opt/lvs/scripts/rs_state_change.sh 172.17.2.236 172.18.2.198 0 80 8080"
    #后端服务器权重
	weight 1
    #后端服务器健康检查
	TCP_CHECK {
	    #链接超时时间
        connect_timeout 10
		#重试次数
        nb_get_retry 3
		#重试间隔
        delay_before_retry 3
        #后端服务器检查端口，一旦没有监听就会剔除
		connect_port 8080
    }
}

```

#### 后端服务器需要配置默认网关为LAN_GATEWAY中的VIP

![TIM图片20180613172610](https://qiniu.li-rui.top/TIM%E5%9B%BE%E7%89%8720180613172610.png)

- 模板配置文件

#### nodeA

```bash
grobal_defs {

   router_id NodeA
}

#vrrp_script chk_nginx {  
#    script "/etc/keepalived/nginx_check.sh"   
#    interval 2  
#    weight 2   
#} 

vrrp_sync_group VG1 { 
   group { 
      VI_1 
      VI_GATEWAY 
   } 
} 

vrrp_instance VI_1 {
   state MASTER
   interface eth1
   virtual_router_id 76
   priority 100
   advert_int 1
   authentication {
       auth_type PASS
       auth_pass 1111
   }
unicast_src_ip  172.16.2.225
   unicast_peer {
       172.16.2.226
   }
   
#track_script {
#   chk_nginx
#   }

   virtual_ipaddress {
      116.255.129.17/24 dev eth0 label eth0
   }
    notify_master "/opt/wg.sh"
}

vrrp_instance VI_GATEWAY { 
        state MASTER 
        interface eth1 
        virtual_router_id 77
        priority 100 
        advert_int 1 
        authentication { 
            auth_type PASS 
            auth_pass 2222 
        } 
        virtual_ipaddress { 
            172.16.2.227/24 dev eth1 label eth1:0
        } 
}


virtual_server 116.255.129.17 80 {
delay_loop 15
lb_algo rr
lb_kind NAT
protocol TCP 
real_server 172.16.2.222 80 {
  weight 1
  TCP_CHECK { 
    connect_port 80
	connect_timeout 3
	nb_get_retry 3
	delay_before_retry 4
	} 
}
real_server 172.16.2.223 80 {
  weight 1 
  TCP_CHECK {
    connect_port 80
	connect_timeout 3 
	nb_get_retry 3 
	delay_before_retry 4 
	} 
  }
}

```

#### nodeB

```bash
grobal_defs {

   router_id NodeB
}

#vrrp_script chk_nginx {  
#    script "/etc/keepalived/nginx_check.sh"      
#    interval 2
#    weight 2
#} 


vrrp_sync_group VG1 { 
   group { 
      VI_1 
      VI_GATEWAY 
   } 
} 


vrrp_instance VI_1 {
   state BACKUP
   interface eth1
   virtual_router_id 76
   priority 99
   advert_int 1
   authentication {
       auth_type PASS
       auth_pass 1111
   }

#track_script {
#   chk_nginx
#   }
unicast_src_ip  172.16.2.226
   unicast_peer {
       172.16.2.225
   }   

   virtual_ipaddress {
       116.255.129.17/24 dev eth0 label eth0
   }
   notify_master "/opt/wg.sh"
}

vrrp_instance VI_GATEWAY {
        state MASTER
        interface eth1
        virtual_router_id 77
        priority 99
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass 2222
        }
        virtual_ipaddress {
            172.16.2.227/24 dev eth1 label eth1:0
        }
}



virtual_server 116.255.129.17 80 {
delay_loop 15
lb_algo rr
lb_kind NAT
protocol TCP 
real_server 172.16.2.222 80 {
  weight 1
  TCP_CHECK { 
    connect_port 80
	connect_timeout 3
	nb_get_retry 3
	delay_before_retry 4
	} 
}
real_server 172.16.2.223 80 {
  weight 1 
  TCP_CHECK {
    connect_port 80
	connect_timeout 3 
	nb_get_retry 3 
	delay_before_retry 4 
	} 
  }
}

```

**后端服务器172.16.2.223/222.默认网关设置为172.16.2.227**

### 4. DR直接路由模式

- 概览

客户端发送请求到复杂均衡器，源mac为客户端IP，目标mac为负载均衡器地址

负载均衡器接受请求后，将目标mac更改为后端服务器mac，并进行广播，即负载均衡器和后端服务器，目标IP没有改，所以会通过后端服务器直接发送给客户端
必须在同一物理网络内，因为要对后端服务器进行ARP广播

后端服务器依据mac接受数据包后，将相应报文**直接**返回给客户端，所以要求后端服务器上面必须要要有虚拟IP地址，

后端服务器做arp限制，只响应负载均衡器的请求，后端服务器可以为windows

- 配置

负载均衡器

```bash
#负载均衡器上配置虚拟IP172.16.206.110
#掩码为255.255.255.255每个IP就是一个网段，这样以来在后端服务器器上添加同一个IP地址就不会警告IP冲突。广播地址也是自己
ifconfig eth0:0 172.16.206.110 netmask 255.255.255.255 broadcast 172.16.206.110

#负载均衡器上关闭防火墙
service iptables stop

#创建负载均衡集群，轮询调度
ipvsadm -A -t 172.16.206.110:80 -s rr

#添加后端服务器
ipvsadm -a -t 172.16.206.110:80 -r 172.16.206.128 -g
ipvsadm -a -t 172.16.206.110:80 -r 172.16.206.130 -g

```

后端服务器配置

```bash
#配置虚拟IP
#负载均衡器只是改了目标mac，并没有改动目标IP，后端服务器上配置IP是为了接受负载均衡器的数据包而不是因为自己主机上没有这个IP而去丢弃数据包
ifconfig lo:0 172.16.206.110 netmask 255.255.255.255 broadcast 172.16.206.110

#添加路由规则(非必需)
route add -host 172.16.206.110 dev lo:0

#抑制ARP
#为什么要抑制arp，因为必须要确保vip所映射的mac是负载均衡器的mac而不是后端服务器的mac

#arp_ignore含义
#Linux接受到ARP数据包后，发送响应数据包的级别(0~8)
#0 只要ARP请求数据包的IP在本机上，都发送响应数据包，即使请求的IP不在接受ARP请求的网卡上面也要发送响应
#1 只有ARP响应数据包中的IP在接受请求的网卡上才进行响应
#2 1的基础上还要求发送放的IP地址属于当前网卡的子网
#设置为1就是只有ARP请求中的IP在当前接受响应的网卡上才进行响应
echo "1" > /proc/sys/net/ipv4/conf/lo/arp_ignore

#arp_announce含义
#主机要发送一个数据包之前，数据包的源IP，目的IP，以及源mac都是确定的，但是目标mac不是确定的。
#这就需要主机发送ARP协议去寻找目标mac，但是当主机上面存在多个网卡和多个IP时，
#Linux系统默认的ARP请求的源IP是数据包的源IP，也就是虚拟IP，
#这样的话下一跳路由的arp缓存就会建立虚拟IP和后端某一个服务器的mac映射关系，
#客户端再次发送请求就会只系欸发送到后端服务器从而绕过负载均衡器。
#那该怎么办？
#这个由arp_announce来决定
#0 默认可以使用任意网络接口的IP地址
#2 首先不使用IP数据包的源地址作为arp请求的源地址
#选择主机中的其他能够做出回应ARP请求的网络接口来作为arp源IP
echo "2" > /proc/sys/net/ipv4/conf/lo/arp_announce
echo "1" > /proc/sys/net/ipv4/conf/all/arp_ignore
echo "2" > /proc/sys/net/ipv4/conf/all/arp_announce



```
