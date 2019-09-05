---
title: ansible多文件处理
date: 2019-09-05 12:12:28
tags:
- ansible
---

# 遇到的问题

helm chart里面有子模板的概念，为了批量管理假如每个chart都有一个共同的文件 config.tpl 。那么如何一次性编辑这个文件呢

<!--more-->

# 主要模块

[find](https://docs.ansible.com/ansible/latest/modules/find_module.html)

思路是通过find模块中的正则表达式来匹配要处理文件的路径，然后使用其他模块去处理，比如 template等

find模块有自己的返回值

![find](https://qiniu.li-rui.top/find.png)

# 核心实现

```yaml
- name: get file paths
  find:
    paths: "{{ paths }}"
    patterns: "{{ patterns }}"
    use_regex: yes
    recurse: yes
  #注意阅读find模块的返回值  
  register: file_path

- name: templates file
  template:
    # 如果是chart的文件 注意和 jinja2 变量文件区分开
    variable_end_string: "UU"
    variable_start_string: "UU"
    backup: yes
    src: file-tmp.j2
    dest: "{{ item.path }}"
  with_items: "{{ file_path.files }}"
```

# role

该模块已经创建了role，[欢迎使用](https://galaxy.ansible.com/sunnoy/mutifile_handle)



