---
title: grandpa模块剖析
copyright: true
date: 2019-10-16 17:45:28
categories:
- BlockChain
- Substrate
- base
tags:
---

# 简介

之前已经翻译了部分[Byzantine Finality Gadgets](https://github.com/w3f/consensus/blob/master/pdf/grandpa.pdf)，可以看我之前发布的[拜占庭终结工具](https://munan.tech/2019/09/30/ByzantineFinalityGadgets)。

<!-- more -->

# 相关模块

在Substrate中，关于终结块机制的模块主要有：`core/finality-grandpa`、`srml/grandpa`和`srml/finality-tracker`。其中，`srml/grandpa`管理了GRANDPA的authority集合，`srml/finality-tracker`跟踪块生产者能感知到的最后终结块，`core/finality-grandpa`是GRANDPA的核心代码部分。

# 关键函数

初始时，默认GRANDPA的authority为验证节点集合。在本文中，仅讨论验证节点上的GRANDPA运行流程。

GRANDPA的核心运行函数是`core/finality-grandpa/src/lib.rs`中的`run_grandpa_voter`，以Substrate上的node为例，介绍该函数的调用过程：

![run_grandpa_voter](https://cdn.jsdelivr.net/gh/JNhua/blog_images@master/img/20201029105752.png)

注意，在当前版本e84f6158b383880ba5fc86cb62c9fea10f5f3071中，服务的future统一在service中去执行。使用`service.spawn_essential_task`或者`service.spawn_task`将服务的future发送到通道`to_spawn_tx`中，然后在service中的`poll()`中，循环用`to_spawn_rx`接收所有服务的future，并用`tokio_executor`执行任务，直到退出。

运行`run_grandpa_voter`前，必要的需要实现`block_import`和得到`LinkHalf`，这两个在宏`new_full_start`完成。

在`run_grandpa_voter`中，先建立了`NetworkBridge`，返回服务句柄和一个future——`network_startup`。`network_startup`在`tokio_executor`注册了3个任务：rebroadcast、announce和reporting。接着为`finality_tracker`模块注册了`inherent_data_providers`，并且读取`client.backend().blockchain().info().finalized_num`存入`Inherent_data`。然后如果之前建立的`telemetry_on_connect`不为空，就建立一个`telemetry_task`的future，之后会检查一次，确保不会因为该任务的意外中断而导致grandpa停止，因为最后有`voter_work.select2(telemetry_task)`。

然后用`VoterWork::new()`建立一个`voter_work`的future，并且检查是否建立成功。将其与之前建立的`network_startup`组合起来。

