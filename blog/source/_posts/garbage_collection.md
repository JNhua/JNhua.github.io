---
title: garbage_collection
date: '2020/11/23 19:47:07'
updated: '2020/11/23 19:47:40'
tags: []
category:
  - Java
  - JVM
mathjax: true
abbrlink: 1e17edb
---
# 垃圾回收条件
## 新生代
Minor GC/Young GC的触发条件：新生代内存区满，而又有新进入的新生代对象。此时会将没有被引用的垃圾新生代对象都回收掉。
<!--more-->
## 老年代

当一个新生代对象在多次躲过垃圾回收，则会被转移到老年代。如果老年代也满了，也会触发垃圾回收。

## 注意

垃圾回收针对的是堆中的新生代、老年代，方法区（永久代），不会针对虚拟机栈中的栈帧。方法一旦执行完毕，栈帧出栈，局部变量直接就被清理掉了。

# JVM内存相关核心参数

**-Xms**：Java堆内存的大小

**-Xmx**：Java堆内存的最大大小

**-Xmn**：Java堆内存中的新生代大小，扣除新生代剩下的就是老年代的内存大小了

**-XX:PermSize**：永久代大小

**-XX:MaxPermSize**：永久代最大大小

**-Xss**：每个线程的栈内存大小

> 永久代在1.8中已经移除，替换为元空间，使用本地内存，突破了**-XX:MaxPermSize**的限制。

![jvm_memory](https://cdn.jsdelivr.net/gh/JNhua/blog_images@master/img/20201029105731.jpg)

应用框架设置JVM参数：Spring Boot其实就是启动的时候可以加上JVM参数，Tomcat就是在bin目录下的catalina.sh中可以加入JVM参数。

