---
title: ceph api使用
date: 2018-11-04 12:12:28
tags:
- ceph
---

# ceph api使用

ceph官方提供的rest api有两个，一个是ceph提供的ceph rest api工具，一个是mgr提供的rest api模块，前一个在12.2.5已经废弃，13版本将会移除，以后主要推行mgr提供的rest api模块。[见](https://ceph.com/releases/v12-2-5-luminous-released/)

虽然ceph rest api工具在未来会被废弃，但是现阶段还是有很多周边工具在使用，比如metricbeat v6.3中的[ceph模块](https://www.elastic.co/guide/en/beats/metricbeat/6.3/metricbeat-module-ceph.html)就是用的该工具来获取监控信息

本文就来看一下这两个api的使用。

<!--more-->

## ceph rest api工具

ceph rest api本身是一个WSGI应用，默认监听5000端口，通过URL来返回json或xml格式，通过网页访问`127.0.0.1:5000/api/v0.1`来获取详细的api信息。

### 工具安装

要使用ceph rest api工具已经包含在ceph-common里面，因此需要安装该包，一般安装ceph这个包都会安装

```bash
yum install ceph-common -y
```
### 基本使用

#### 为该api创建用户

ceph api rest工具默认使用client.restapi用户的权限，可以新建用户然后分配给rest工具来使用，也可以对用户client.restapi的权限来配置

修改client.restapi权限

```bash
ceph auth get-or-create client.restapi mds 'allow' osd 'allow *' mon 'allow *' > /etc/ceph/ceph.client.restapi.keyring
```

加入将下面加入ceph.conf里面

```bash
[client.restapi]
log file = /var/log/ceph/ceph.restapi.log
keyring = /etc/ceph/ceph.client.restapi.keyring
#可以更改端口，默认 0.0.0.0:5000
public addr = 173.17.12.8:2325
#日志级别，可选critical, error, warning(默认), info, debug
restapi log level = warning
```

#### 启动工具

```bash
#默认就是-c /etc/ceph/ceph.conf --cluster ceph
#可以直接ceph-rest-api启动
nohup ceph-rest-api > /var/log/ceph-rest-api &> /var/log/ceph-rest-api-error.log &
```

#### 使用

查看cursh规则

```bash
curl  192.168.66.110:5000/api/v0.1/osd/crush/rule/list

replicated_rule
```

## mgr rest 模块

### 基本配置

RESTful插件可以提供一些监控信息和进行一些ceph操作

#### 有关mgr restful操作

```bash
#查看插件
ceph mgr module ls
{
    "enabled_modules": [
        "dashboard",
        "restful",
        "status"
    ],
    "disabled_modules": [
        "balancer",
        "influx",
        "localpool",
        "prometheus",
        "selftest",
        "zabbix"
    ]
}

#启用插件
ceph mgr module enable <module>
#禁用插件
ceph mgr module disable <module>
#查看mgr服务
ceph mgr services
{
    "dashboard": "http://matrix_03:7000/",
    "restful": "https://173.20.1.103:8003/"
}

```

#### restful key的命令

该命令主要管理认证key使用

```bash
#创建key
ceph restful create-key <key_name>
#创建自签证书
ceph restful create-self-signed-cert
#删除key
ceph restful delete-key
#列出key
ceph restful list-keys
#重启api服务
ceph restful restart
```

#### ceph config-key命令

该命令主要针对restful服务配置使用，配置完成后注意使用`systemctl restart ceph-mgr@matrix_03`使配置生效

```bash
#查看config-key中的键值对
config-key dump
#查看config-key中的key
ceph config-key ls
[
    "initial_mon_keyring",
    "mgr/restful/keys/admin",
    "mgr/restful/keys/dong",
    "mgr/restful/matrix_03/crt",
    "mgr/restful/matrix_03/key",
    "mgr/restful/matrix_03/server_addr",
    "mgr/restful/matrix_03/server_port"
]
#设置相关key，比如设置api的IP和端口
ceph config-key set mgr/restful/$name/server_addr $IP
ceph config-key set mgr/restful/$name/server_port $PORT
#获取相关key
config-key get <key> 
#查看是否存在key
config-key exists <key>
#更多命令查看
ceph config-key help
```


### 配置api接口

#### 启用插件

```bash
ceph mgr module enable restful
```

#### ssl配置

该接口需要ssl配置，可以使用自有证书也可以使用自签证书

##### 自有证书

```bash
ceph config-key set mgr/restful/node1/crt -i restful.crt
ceph config-key set mgr/restful/node1/key -i restful.key
#如果每台mgr证书都一样则去掉node1
ceph config-key set mgr/restful/crt -i restful.crt
ceph config-key set mgr/restful/key -i restful.key
```

##### 使用自签证书

```bash
ceph restful create-self-signed-cert
```

##### 创建用户

会返回密码
```bash
ceph restful create-key admin
```

#### 基本使用

##### 基本文档

在浏览器使用链接`https://ceph-mgr/:8003/doc`查看基本文档。[更多使用文档](https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/3/html-single/ceph_management_api/index#how-can-i-view-crush-rules)

##### 浏览器内查看

也可以在浏览器进行相关信息查看

比如查看cursh规则需要访问`https://192.168.66.112:8003/crush/rule`，然后再弹出框输入用户名和密码，就是命令`ceph restful create-key admin`创建的key和密码

##### curl查看

```bash
#自签证书需要--insecure
curl --silent --insecure --user dd 'https://<ceph-mgr>:8003/crush/rule'

curl --silent --insecure --user dd 'https://192.168.66.112:8003/crush/rule'
Enter host password for user 'dd':
[
    {
        "max_size": 10,
        "min_size": 1,
        "osd_count": 9,
        "rule_id": 0,
        "rule_name": "replicated_rule",
        "ruleset": 0,
        "steps": [
            {
                "item": -1,
                "item_name": "default",
                "op": "take"
            },
            {
                "num": 0,
                "op": "chooseleaf_firstn",
                "type": "host"
            },
            {
                "op": "emit"
            }
        ],
        "type": 1
    }
]
```

##### python查看

```bash
#自签证书需要verify=False
$ python
>> import requests
>> result = requests.get('https://<ceph-mgr>:8003/crush/rule', auth=("<user>", "<password>"), verify=False)
>> print result.json()
```

## mgr dashboard 模块

### 基本配置

#### 开启模块

默认为监听http://0.0.0.0:7000，没有用户认证也没有管理对象网关

```bash
ceph mgr module enable dashboard
```
在配置文件中初始化开启

```conf
[mon]
mgr initial modules = dashboard restful
```

#### 一些配置

也可以自签证书

```bash
ceph dashboard create-self-signed-cert
```

使用自有证书和restful插件一样

```bash
ceph config-key set mgr mgr/dashboard/crt -i dashboard.crt
ceph config-key set mgr mgr/dashboard/key -i dashboard.key
```
也可以更改IP和端口

```bash
ceph config-key set mgr/dashboard/$name/server_addr $IP
ceph config-key set mgr/dashboard/$name/server_port $PORT
```

为dashboard创建用户名和密码

```bash
ceph dashboard set-login-credentials <username> <password>
```

更改配置除了重启mgr生效意外还有

```bash
ceph mgr module disable dashboard
ceph mgr module enable dashboard
```

### 管理对象网关

#### 创建对象网关用户

```bash
#改命令会返回access_key和secret_key
radosgw-admin user create --uid=<user_id> --display-name=<display_name> --system
```

#### 对接

dashboard会自动寻找对象网关地址和端口

```bash
ceph dashboard set-rgw-api-access-key <access_key>
ceph dashboard set-rgw-api-secret-key <secret_key>
```

#### 其他的一些配置

```bash
ceph dashboard set-rgw-api-host <host>
ceph dashboard set-rgw-api-port <port>
ceph dashboard set-rgw-api-scheme <scheme>  # http or https
ceph dashboard set-rgw-api-admin-resource <admin_resource>
ceph dashboard set-rgw-api-user-id <user_id>
ceph dashboard set-rest-requests-timeout <seconds> #默认40s
```

## prometheus模块

用来让prometheus收集监控信息

### 开启模块

```bash
ceph mgr module enable prometheus
```

开启后会默认开启9283监听

### prometheus配置

```bash
  - job_name: 'ceph'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['192.168.66.112:9283']
```







