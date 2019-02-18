---
title: ansible常用模块
date: 2018-11-29 12:12:28
tags:
- ansible
---

# ansible常用模块

## 基本介绍

ansible中执行操作最终是落到具体模块上的，ansible提供了非常繁多的模块可供使用，覆盖了系统和应用的方方面面，这里介绍几个常用的模块

模块列表[在此](https://docs.ansible.com/ansible/latest/modules/list_of_all_modules.html)

![mode](https://qiniu.li-rui.top/mode.png)

<!--more-->

### 忽略错误

```yml
- name: this will not be counted as a failure
  command: /bin/false
  ignore_errors: yes
```

### 自定义错误

下面实例当register出现所需字符串时就报错

```yaml
- name: Fail task when the command error output prints FAILED
  command: /usr/bin/example-command -x -y -z
  register: command_result
  #当变量内容出现FAILED的时候就会返回执行fail信息
  failed_when: "'FAILED' in command_result.stderr"
```

### debug模块

针对playbook进行调试

```yml
- debug:
    msg: "System {{ inventory_hostname }} has gateway {{ ansible_default_ipv4.gateway }}"
  when: ansible_default_ipv4.gateway is defined
```

### 用户和用户组

下面为添加用户和用户组，并创建文件

```yml
- hosts: node1
  tasks:
  - name: create new group
    group:
        name: gtest
        state: present
  - name: create new user
    user:
        name: utest
        group: gtest
        password: xinyue
        state: present
  - name: create new file
    file:
        path: /var/log/test
        state: touch
        owner: utest
        group: gtest
        mode: 0775
```

### 在本地运行

```yml
- name: Send summary mail
  local_action:
    module: mail
    subject: "Summary Mail"
    to: "{{ mail_recipient }}"
    body: "{{ mail_body }}"
    run_once: True
```

### task异步

```yml

```

### 检测远程主机

在规定时间内等待远程主机上线

```yml
- name: Wait for system to become reachable
  wait_for_connection:
    connect_timeout: 4
    delay: 600
    sleep: 5
    timeout: 2000
```

### task异步

```yml
- name: 轮询等待flannel 运行，视下载镜像速度而定
  shell: "{{ bin_dir }}/kubectl get pod -n kube-system -o wide|grep 'flannel'|grep ' {{ inventory_hostname }} '|awk '{print $3}'"
  register: pod_status
  until: pod_status.stdout == "Running"
  retries: 15
  delay: 8

#又一个例证
- name: ping hosts
  ping: 
  async: 3000
  poll: 0
  register: ping_sleeper
  
#等待状态后操作
- name: check system install status
  shell: cobbler status
  register: system_status
  until: system_status.stdout.find('installing') == -1 and system_status.stdout.find('unknown') == -1 and system_status.stdout.find('stalled') == -1 and system_status.stdout.find('finished') != -1 
  retries: 5000
  delay: 2
```

### notify

主要用来更改配置后重启服务

```yml
- name: config keepalived file
  template:
    src: keepalived.conf
    dest: /etc/keepalived/dong.conf
  notify:
     - restart keepalived
#在handler目录
- name: restart keepalived
  service:
    name: keepalived
    state: restarted
```

### tags

添加tags

```yml
- name: config keepalived file
  template:
    src: keepalived.conf
    dest: /etc/keepalived/dong.conf
  notify:
     - restart keepalived
  tags: test
  #数组也可以
  tags: [ 'web', 'foo' ]
```

使用tags

```bash
#列出tags
ansible-playbook -i ss  lvs-ha.yml --list-tags
#指定tag
ansible-playbook ... --tags "notification"
#跳过tag
--skip-tags
```

几种特殊tags

- always 总是会运行除非使用`obbler always`才会跳过
- never 总是不运行，除非使用进行执行
- tagged 只运行tag的task
- untagged 只运行没有tag的task
- all 运行所有的task

### yml文件引用

```yml
- name: include checks/check_firewall.yml
  include: checks/check_firewall.yml
  when:
    - check_firewall
  static: False
```

### shell、command、script、raw

- shell 执行远程主机的shell/python脚本
- command 在远程主机上执行命令

- script 将本地脚本拷贝到远程主机上执行
- raw 类似于command模块、支持管道传递


### fetch

将远程主机的文件复制到本地，只可以是文件而不是文件夹

```bash
 ansible -i hosts.json lvss[0] -m fetch -a "src=/opt/lvs.tar.gz dest=./"
```

### 本地的playbook

创建playbook文件

```yml
- hosts: 127.0.0.1
  connection: local
```

执行

```bash
#直接
ansible-playbook playbook.yml 

#也可以
ansible-playbook playbook.yml --connection=local
```