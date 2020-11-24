---
title: clion远程调试rust
date: '2020/11/23 19:39:43'
updated: '2020/11/23 19:40:19'
tags: []
category:
  - 编程
  - 编程
mathjax: true
---
# 目标
在CLion下远程调试Rust代码。
<!--more-->
## 环境
* 本地Mac OS，远程Ubuntu。
* IDE：CLion2019.3.4

# 机器设置

## 远程端

需要打开端口ssh的TCP 22，gdbserver的TCP 2345。

需要安装rust与gdb。

## 本地端

安装好Clion，rust插件与rust。

### CLion配置

1. Tools -> Deployment -> Configuration

![image-20200907102258454](https://cdn.jsdelivr.net/gh/JNhua/blog_images@master/img/20201029105330.png)

![image-20200907102510024](https://cdn.jsdelivr.net/gh/JNhua/blog_images@master/img/20201029105403.png)

2. 在Mapppings中添加映射。
   1. 添加代码文件夹的映射；
   2. 添加本地target_remote 到 远程target的映射，为了让Linux的target文件与本地Mac OS的target文件区分开。所以，还需要在Excluded Paths中，把本地的target文件夹排除掉。

![image-20200907103320500](https://cdn.jsdelivr.net/gh/JNhua/blog_images@master/img/20201029105417.png)

![image-20200907103404223](https://cdn.jsdelivr.net/gh/JNhua/blog_images@master/img/20201029105426.png)

3. 上传代码文件，在项目的根目录点击右键，然后上传

![image-20200907103748150](https://cdn.jsdelivr.net/gh/JNhua/blog_images@master/img/20201029105437.png)

## 在远程端编译

* `cargo build`生成target文件夹。
* 开启gdbserver，例如：`gdbserver 0.0.0.0:2345 ./target/debug/cita-executor`。
* 本地同步远程端，把target文件夹下载到本地的target_remote文件夹。
  * **Tools→Deployment→Browse Remote Host**，找到target文件夹，右键选择Dowanload from here。

## 调试设置

添加GDB Remote Debug，就可以本地测试啦。