---
title: jenkins时区修改
date: 2019-12-03 12:12:28
tags:
- jenkins
---

# Jenkins时区

Jenkins时区默认是utc

<!--more-->

# 时区修改

## 官方指引

https://wiki.jenkins.io/display/JENKINS/Change+time+zone

### 使用java

需要重启

```bash
java -Dorg.apache.commons.jelly.tags.fmt.timeZone=Asia/Shanghai
```

### 使用控制台

Manage Jenkins -> Script Console 然后输入下面字符后 run

```bash
System.setProperty('org.apache.commons.jelly.tags.fmt.timeZone', 'Asia/Shanghai')
```

