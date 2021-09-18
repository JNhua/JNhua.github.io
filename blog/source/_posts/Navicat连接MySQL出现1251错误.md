---
title: Navicat连接MySQL出现1251错误
date: '2021/9/18 11:24:23'
updated: '2021/9/18 11:27:15'
tags: []
category:
  - 编程
  - database
mathjax: true
toc: false
---
# 1251错误
![1251error](https://cdn.jsdelivr.net/gh/JNhua/blog_images/img/20210918112513.png)
<!--more-->
# 原因分析
出现这个原因是mysql8 之前的版本中加密规则是mysql_native_password,而在mysql8之后,加密规则是caching_sha2_password, 解决问题方法有两种,一种是升级navicat驱动,一种是把mysql用户登录密码加密规则还原成mysql_native_password. 
# 解决
采用第二种方式

ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'password'; #修改加密规则 

ALTER USER 'root'@'localhost' IDENTIFIED BY 'password' PASSWORD EXPIRE NEVER; #更新一下用户的密码 

FLUSH PRIVILEGES; #刷新权限 

> 'root'   为你自己定义的用户名
'localhost' 指的是用户开放的IP
'password' 是你想使用的用户密码
