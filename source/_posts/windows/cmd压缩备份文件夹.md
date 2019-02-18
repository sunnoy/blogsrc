---
title: cmd压缩备份文件夹
date: 2018-07-30 14:12:28
tags:
- Windows
- cmd
- 文件服务

---

## 压缩

### 下载

使用开源7z来压缩

```cmd
#下载地址
https://www.7-zip.org/download.html
```

我们要使用命令行，所以下载这个

![7z下载](https://qiniu.li-rui.top/7z下载.png)

<!--more-->

### 解压

解压后就可以使用命令行了，下面是主要文件，其他的就可以删除了

![解压](https://qiniu.li-rui.top/解压.png)

## 批处理编写

### 脚本内容

我们的目的是压缩一个文件夹，然后以当天的日期命名

**创建批处理文件**

```cmd
set dir=E:\chromeDownload\7z1604-extra
set time=%date:~0,4%-%date:~5,2%-%date:~8,2%-bak
%dir%\7za.exe a %time%.zip E:\chromeDownload\7z1604-extra\Far\*
```

### 解释

首先创建，执行程序的位置变量，这个加入环境变量也可以

然后给出时间格式为`2018-07-30`，`:~0,4`用来对字符串进行切割

最后执行压缩操作

## 7z的其他命令

```cmd
# 指定若干文件
7za.exe a test.zip a.txt b.txt 
# 压缩文件夹
7za.exe a test.zip f:/test/**

# -o表示输出目录，注意其与目录路径之间没有空格
7za.exe x test.zip -of:\test 
# 假如输出文件夹有空格，用引号包裹
7za.exe x test.zip -o"f:\test abc" # 假如输出文件夹有空格，用引号包裹
```

## 保留多少天之内的文件

### 关键代码

```cmd
forfiles /p E:\chromeDownload /m "20*" /d -31  /c "cmd /c if @isdir==TRUE (rmdir /q /s @path) else (del /f @path)"
```

### 使用案例

![使用案例](https://qiniu.li-rui.top/使用案例.png)

### 计划任务之windows server 2012

#### 打开计划任务程序

![任务计划1](https://qiniu.li-rui.top/任务计划1.png)

#### 配置

![任务计划2](https://qiniu.li-rui.top/任务计划2.png)




