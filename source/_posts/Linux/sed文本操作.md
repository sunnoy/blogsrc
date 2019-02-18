---
title: sed文本操作
date: 2018-11-04 12:12:28
tags:
- linux
---
## sed文本操作

### 1. 替换整行

- 行中目标字符串

```bash
sed -i "s/.*mongohost = .*/  mongohost = '$1'/" /opt/shell/pycron/mongo.py
```
<!--more-->
### 去掉空行和注释

```bash
sed -i -c -e '/^$/d;/^#/d' config_file
cat /etc/vsftpd/vsftpd.conf | grep -v ^# | grep -v ^$
```

### 去掉xml注释和空行

```bash
#!/bin/bash
#删除ibaits下xml文件的注释与空行
#find -name '*.xml'|while read f
#do
#  echo $f
#  cat $f|sed  's/<!--/\n<!--\n/'|sed  's/-->/\n-->\n/'|sed  '/<!--/,/-->/ d'|sed  '/-->/ d'|sed '/^\s*$/d'  >$f
#done
for f in '*.xml'
do
 # echo $f
  sed  -i 's/<!--/\n<!--\n/' $f
  sed  -i 's/-->/\n-->\n/' $f
  sed  -i '/<!--/,/-->/ d' $f
  sed  -i '/-->/ d' $f
  sed -i '/^\s*$/d' $f
done

```
