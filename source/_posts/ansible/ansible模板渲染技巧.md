---
title: ansible模板渲染技巧
date: 2018-11-29 12:12:28
tags:
- ansible
---

# ansible模板渲染技巧

本文主要介绍集中自己探索的jinja2模板非常规使用

## 中间变量桥梁

<!--more-->

### 需求

需求：在使用ceph部署的时一些变量需要定义在主机变量内，即`host_vars/主机名.yml`内

需要达到如下效果，在发现有的主机是mon角色的时候就添加`monitor_address`

```yml
osd_scenario: collocated
osd_objectstore: bluestore
devices:
  - /dev/sdb
  - /dev/sdc
  - /dev/sdd
monitor_address: 173.20.1.101
radosgw_address: 173.16.1.101
```

### 操作

### 定义变量格式

```yml
# group_vars/all.yml
hosts:
  matrix01: 
    ip: 1
    ceph_mon: 1
```

### 创建中间变量，并引入

首先创建`hosts`变量的”副本“

```yml
- name: create var file
  template:
    src: var.yml.j2
    dest: /opt/var.yml

```

模板文件

```
---
{% for host, attr in hosts.iteritems() %}
{# {% set net_ip_num = common.large_ip - loop.index + 1 %} #}
{% set net_ip_num = attr.ip %}
{{ host }}: {{ net_ip_num }}
{% endfor %}
```

将变量文件引入，并注册到变量stuff

```yml
#引入变量文件
- name: reg vars
  include_vars:
    file: /opt/var.yml
    #name: stuff
  register: stuff
```

### 渲染出host变量文件

```yml
- name: create host yml files
  template:
    src: osd_hosts.j2
    dest: "host_vars/{{ item.key }}.yml"
  when: item.value.ceph_osd or item.value.ceph_mon or item.value.ceph_rgw
  loop: "{{ hosts|dict2items }}"
```

模板文件，给文件分别对hosts变量和stuff进行遍历，当存在ceph_mon时就加入monitor地址

```
{% if item.value.ceph_mon %}
{% for kk, vv in stuff.ansible_facts.iteritems()  %}
{%if item.key == kk %} 
monitor_address: 173.20.1.{{ vv }}
{% endif %}
{% endfor %} 
{% endif %}
```

## host动态生成

ceph ansible安装中需要自动生成hosts.ini文件

### 变量

仍然使用前面的

```yml
hosts:
  matrix01: 
    ip: 1
    ceph_mon: 1
```

### 渲染模板

模板

```
[mons]
{% for host, attr in hosts.iteritems() %}
{% if attr.get('ceph_mon') %}
{{ host }}
{% endif %}
{% endfor %}

```

开始渲染

```yml
- name: create host file
  template:
    src: hosts.ini.j2
    dest: hosts.ini
```

渲染后进行刷新ansible host缓存，主要使用`meta: refresh_inventory`元数据

```yml
- hosts: all
  gather_facts: no
  tasks:
    - name: refresh inventory
      meta: refresh_inventory
    - name: Wait for system to become reachable
      wait_for_connection:
        connect_timeout: 4
        delay: 180
        sleep: 2
        timeout: 1000
```

## 多数组渲染

### 需求

一次性加入keepalived后端服务器

### 知识准备

在jina2的for循环中一些变量是可以直接使用的

![loop](https://qiniu.li-rui.top/loop.png)

### 构造变量

virtual_server可以很多个，每个有很多real server

```yml
virtual_server: 
  - ip: 192.168.66.201
    port: 80
    protocol: TCP
    delay_loop: 6
    persistence_timeout: 60
    lb_algo: rr
    lb_kind: NAT
    real_server:
      ip:
        - 11.11.11.12
        #- 127.0.0.5
      port:
        - 80
        #- 66
      connect_port:
        - 80
        #- 66
      connect_timeout:
        - 5
        #- 6
      nb_get_retry:
        - 3
        #- 4
      delay_before_retry:
        - 5
```

### 渲染模板

```yml
- name: Configure keepalived
  template:
    src: keepalived.conf.j2
    dest: "{{ keepalived_config_file_path }}"
  tags:
    - keepalived-config-t
  notify:
    - restart keepalived
```

模板文件

```
{% for vs in virtual_server %}
virtual_server {{ vs.ip }} {{ vs.port }} {
    delay_loop {{ vs.delay_loop }}
    lb_algo {{ vs.lb_algo }}
    lb_kind {{ vs.lb_kind }}
    protocol {{ vs.protocol }}
    persistence_timeout {{ vs.persistence_timeout }}
{% for rs in vs.real_server.ip %}
   
  real_server {{ vs.real_server.ip[loop.index0] }} {{ vs.real_server.port[loop.index0] }} {
  
{% if vs.real_server.connect_port is defined %}  
  TCP_CHECK {
	connect_port {{ vs.real_server.connect_port[loop.index0] }}
	connect_timeout {{ vs.real_server.connect_timeout[loop.index0] | default(5) }}
	nb_get_retry {{ vs.real_server.nb_get_retry[loop.index0] | default(3) }}
	delay_before_retry {{ vs.real_server.delay_before_retry[loop.index0] | default(3) }}
{% endif %}

     }
  }
{% endfor %}
}
{% endfor %}
```









