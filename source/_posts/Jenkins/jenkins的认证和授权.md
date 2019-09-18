---
title: jenkins安装
date: 2018-11-04 12:12:28
tags:
- jenkins
---

# 认证和授权

Jenkins的认证授权在安装策略里面(Configure Global Security->Access Control)，分为两种

- Security Realm 告诉Jenkins环境如何以及从何处获取用户(或身份)信息。 也别称为 "认证"。
- Authorization 配置，他告诉 Jenkins 环境， 哪些用户和/或组可以访问Jenkins的哪些方面, 以及何种程度。

<!--more-->

# Security Realm

Security Realm 是完成认证功能的。主要包含两类Jenkins自己的数据库以及委派第三方的工具实现。默认有四个认证插件

## Delegate to servlet container

通过其他的servlet container来实现，比如使用blueocean的java应用来实现

## LDAP

通过LDAP完成认证系统

## Unix user/group database

通过Linux的用户和用户组来完成认证

## Jenkins’ own user database

使用Jenkins自己的数据库来存储用户信息

本次测试使用该选项

# Authorization

Jenkins的授权默认也包含了几种

## Anyone can do anything

没有授权机制，每个人都是管理员

## Legacy mode

一个用户拥有 "admin" 角色, 就会被授予对系统的完全控制, 负责 (包括匿名用户) 只会拥有读访问权限。不建议使用该策略

## Logged in users can do anything

每个登陆的用户都是管理员的权限

## Matrix-based security

Matrix-based security主要用来分配权限和用户的匹配，因此要求要使用 Jenkins’ own user database 的认证方式

## Project-based Matrix Authorization Strategy

是Matrix-based security的拓展，可以根据项目来绑定不通的用户

## Role-based Authorization Strategy

本文使用 Role-based Authorization Strategy 的插件，需要安装

### 详细使用

详细使用请[参考该博文](https://blog.csdn.net/wanglei_storage/article/details/78339409)

需要指出的是创建Project roles的时候正则表达式(Pattern)填写的时候如果是多个正则表达的条件可以通过`|`来合并

```bash
one.*|two.*
```

匹配以one和two开头的所有的项目的集合

# 从Delegate to servlet container 切换Jenkins’ own user database

Allow users to sign up 不要选，切换后会让创建admin账户


