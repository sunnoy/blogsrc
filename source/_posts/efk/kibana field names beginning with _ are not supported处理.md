---
title: kibana field names beginning with _ are not supported处理
date: 2019-06-27 12:12:28
tags:
- efk
---

 _ 开头的字段名称在elasticsearch里面是保留字段，在kibana中是不被支持的，尤其是获取systemd的日志

 # 用到的仓库

[fluent-plugin-systemd](https://github.com/reevoo/fluent-plugin-systemd)

<!--more-->

# 添加配置

```yaml
<source>
  @type systemd
  tag kubelet
  path /var/log/journal
  matches [{ "_SYSTEMD_UNIT": "kubelet.service" }]
  read_from_head true

  <storage>
    @type local
    path /var/log/fluentd-journald-kubelet-cursor.json
  </storage>

  <entry>
  #关键配置
    fields_strip_underscores true
    fields_lowercase true
  </entry>
</source>

<match kubelet>
  @type stdout
</match>

<system>
  root_dir /var/log/fluentd
</system>
```