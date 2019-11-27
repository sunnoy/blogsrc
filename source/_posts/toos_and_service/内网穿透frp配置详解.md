---
title: 内网穿透frp配置详解
date: 2019-02-20 12:12:28
tags:
- linux
---

# 内网穿透

![frp](https://qiniu.li-rui.top/frp.jpg)

首先考虑这样一个场景：如何在公网上登陆到家里的路由器？现在运营商在我们办理宽带时分配的都是内网地址，一般是10开头，在公网上是不可以主机访问的。

<!--more-->

那么frp之类的工具是如何解决这个问题的呢？他们采用C/S架构，服务端在公网上，客户端在内网中。客户端去连接服务端，然后把内网的服务端口通过加密通道给服务端，然后服务在公网的服务器上开启一个端口和内网的服务端口形成一个映射关系。比如当我们访问服务端的6000端口就是去访问内网的22端口。

[frp仓库地址](https://github.com/fatedier/frp)，frp使用主要是对服务端或者客户端配置文件的编写。[release页面](https://github.com/fatedier/frp/releases)中下载相应平台的包，目录内

- frps为服务端启动文件
- frpc为客户端启动文件

frp当前版本为v0.24.1

# frp服务端

## 配置文件解析

```ini
[common]
#服务端监听地址和端口，该端口是和客户端通信的端口，底层通信端口，tcp
#bind_addr为0.0.0.0含义就是服务端服务器上的所有IP都可以监听外部连接请求
#如果需要配置单个就将服务器的某一个公网IP写在bind_addr
bind_addr = 0.0.0.0
bind_port = 7000

#可以使用nat打洞，进行点对点的连接，这时候数据通信不仅过frp服务端
bind_udp_port = 7001

#可以使用底层通信端口为kcp
kcp_bind_port = 7000

#考虑到这个场景：内网的的数据库是个备库，frps这边要把本地的数据库备份到内网机器上，在此使用proxy_bind_addr = 127.0.0.1 就可以达到这个效果，还很安全
proxy_bind_addr = 127.0.0.1

#域名访问内网的web服务，需要将域名解析到服务器，A记录或者CNAME记录
vhost_http_port = 80
vhost_https_port = 443

#http相应超时，默认60s
vhost_http_timeout = 60

#frpweb统计界面
dashboard_addr = 0.0.0.0
dashboard_port = 7500

#web统计界面认证信息
dashboard_user = admin
dashboard_pwd = admin

#frp服务端日志文件
log_file = ./frps.log

#日志等级 trace, debug, info, warn, error
log_level = info

#日志保留天数
log_max_days = 3

#frp服务端和客户端通过bind_port端口进行认证的token，服务端和客户端都要一直
token = 12345678

#服务器上可以用映射的端口
allow_ports = 2000-3000,3001,3003,4000-50000


#默认情况下：当用户请求建立连接后，frps 才会请求 frpc 主动与后端服务建立一个连接 
#如果由大量短连接的情况呢，是不是很和耗时？因此，该字段会在和frp客户端连接后，主动建立max_pool_count个连接，当有用户来访问业务的是时候就会从该连接池内取出连接来用，使用于大量短连接的情况
max_pool_count = 5

#限制frpc使用服务端端口的数量。0为不限制
max_ports_per_client = 0

#该功能需要泛解析，即把*.frps.com A记录到frps在的服务器上，然后frpc就可以只配置subdomain = test 就可以使用 test.frps.com来使用，多人使用很方便
subdomain_host = frps.com

#启动tcp多路复用
tcp_mux = true
```

# frpc客户端

## 配置解析common字段

```ini
[common]
#和frps底层通信的信息，和bind_addr = 0.0.0.0 bind_port = 7000 保持一致
server_addr = 0.0.0.0
server_port = 7000

#frpc通过代理去连接frps。http或socks5代理方式
#http_proxy = http://user:passwd@192.168.1.128:8080
http_proxy = socks5://user:passwd@192.168.1.128:1080

#日志文件
log_file = ./frpc.log

#日志等级 trace, debug, info, warn, error
log_level = info

#日志最大保留天数
log_max_days = 3

#和frps底层通信认证token和frps一致
token = 12345678

#热加载frpc配置使用 frpc reload -c ./frpc.ini
admin_addr = 127.0.0.1
admin_port = 7400
admin_user = admin
admin_pwd = admin

#和frps刚开始通信时建立的连接数量
pool_count = 5

#开启tcp多路复用 和frps一致
tcp_mux = true

#加入frpc标识，下面的模块使用 如 your_name.ssh
user = your_name

#frpc和frps登陆失败后是否继续持续登陆
login_fail_exit = true

#和frps通信的底层协议 tcp kcp websocket,  默认是 tcp
protocol = tcp

#frpc使用的dns服务器
# dns_server = 8.8.8.8

#frpc中启动的代理模块，模式都启动
# start = ssh,dns

#ssh代理模块
[ssh]
...
...

#dns代理模块
[dns]
...
...
```

## 代理模块配置

```ini

#代理模块名称，可以存在多个代理模块
[ssh]

#代理模块类型
#支持 tcp | udp | http | https | stcp | xtcp, 默认是is tcp
type = tcp

#客户端本地多要代理的地址和端口
local_ip = 127.0.0.1
local_port = 22

#对本地unix监听
#添加插件
plugin = unix_domain_socket
#sock位置
plugin_unix_path = /var/run/docker.sock

#底层通信中是否加密 默认不加密
use_encryption = false
#底层通信信息是否压缩
use_compression = false

#映射到frps上的远程端口，0为随机端口，由frps来指配
remote_port = 6001

#端口组映射,需要添加“range:”前缀 tcp udp都可以
[range:abc]
local_port = 6010-6020,6022,6024-6028
remote_port = 6010-6020,6022,6024-6028

#frpc的负载均衡，就是说一个frps映射多个frpc本地端口，
[test1]
type = tcp
local_port = 8080
#remote_port要一致
remote_port = 80
#group group_key 要一致
group = web
group_key = 123
[test2]
type = tcp
local_port = 8081
#remote_port要一致
remote_port = 80
#group group_key 要一致
group = web
group_key = 123

#frpc健康检查
#支持tcp和http
health_check_type = tcp
#连接本地端口服务超时
health_check_timeout_s = 3
#失败多少次就剔除该代理本地端口
health_check_max_failed = 3
#检查频率
health_check_interval_s = 10



#http基本配置
[web01]
type = http
local_ip = 127.0.0.1
local_port = 80

#加入http认证
http_user = admin
http_pwd = admin

#泛解析使用
subdomain = web01
#单个域名使用
custom_domains = web02.yourdomain.com

#web02.yourdomain.com/pic 到这个服务
locations = /pic,/abc
#修改请求头，web02.yourdomain.com会在访问本地网络的时候修改为example.com
host_header_rewrite = example.com
#header_开头后面的东东会添加到http请求头中
header_X-From-Where = frp

#http健康检查
health_check_type = http
#请求健康检查的位置
health_check_url = /status
health_check_interval_s = 10
health_check_max_failed = 3
health_check_timeout_s = 3

#frpc做反向代理，也就是说外网的机器可以连接代理，然后去访问内网的东东
[plugin_http_proxy]
type = tcp
remote_port = 6004
plugin = http_proxy
#代理到本地认证
plugin_http_user = abc
plugin_http_passwd = abc
#反向代理还可以是sock方式
[plugin_socks5]
type = tcp
remote_port = 6005
plugin = socks5
#代理到本地认证
plugin_user = abc
plugin_passwd = abc


#创建一个frpc的文件访问服务
[plugin_static_file]
type = tcp
remote_port = 6006
plugin = static_file
plugin_local_path = /var/www/blog
#前缀http://x.x.x.x:6000/static/
plugin_strip_prefix = static
#http基本认证
plugin_http_user = abc
plugin_http_passwd = abc


#stcp可以让自定的人去访问我们所暴露的网络服务，前提是访问我们服务的人也要运行一个frpc服务
#我们的配置
[secret_sshaaa]
type = stcp
# 只有 sk 一致的用户才能访问到此服务
sk = abcdefg
local_ip = 127.0.0.1
local_port = 22

#另一个可以访问我们服务的货的配置
[secret_ssh_visitor]
type = stcp
# stcp 的访问者
role = visitor
# 要访问的 stcp 代理的名字，没有端口标识，使用代理模块名称来标识
server_name = secret_sshaaa
sk = abcdefg
# 绑定本地端口用于访问 ssh 服务，即目的端口是6000
bind_addr = 127.0.0.1
bind_port = 6000


#nat打洞 请注意 一些nat类型不可以被打洞，frps充当STUN
#类似于stcp，只是流量不经过frps，是单方向上的打洞
#我们配置
# frpc.ini
[p2p_ssh]
type = xtcp
# 只有 sk 一致的用户才能访问到此服务
sk = abcdefg
local_ip = 127.0.0.1
local_port = 22

#另一个想访问p2p_ssh服务的货的配
[p2p_ssh_visitor]
type = xtcp
# xtcp 的访问者
role = visitor
# 要访问的 xtcp 代理的名字
server_name = p2p_ssh
sk = abcdefg
# 绑定本地端口用于访问 ssh 服务
bind_addr = 127.0.0.1
bind_port = 6000

#stcp和xtcp都是将端口绑定到访问者的本地，而不是远程，通过访问本地端口来访问另一边的服务
#因此这样很安全，因为公网上的其他人并无法访问
ssh -oPort=6000 test@127.0.0.1
```

# 命令行使用

```bash
frps \
    --bind_addr 0.0.0.0 \
    --bind_port 1989 \
    --dashboard_addr 0.0.0.0 \
    --dashboard_port 1990 \
    --dashboard_pwd stest \
    --dashboard_user admin \
    --token stest

./frpc tcp \
    --local_port 22 \
    --proxy_name kylin \
    --server_addr xxx.xx.xx.xx:1989 \
    --remote_port 1991 \
    --token stest \
    --uc \
    --ue
```







