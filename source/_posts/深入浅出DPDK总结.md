---
title: DPDK vs 内核协议栈
date: 2025-08-24 23:32:43
updated: 2026-09-29 00:17:35
# Keep the URL of the already-published article when changing its title/date.
permalink: 2025/08/16/深入浅出DPDK笔记/
comments: false
auto_excerpt: true
toc: true
tags: DPDK
categories: 技术学习
description: 简要对比 DPDK 与内核协议栈的开销、优化手段，以及 NAPI 与轮询收包的区别。
---

**DPDK 的核心思路：绕过内核网络协议栈，用轮询、批处理和更可控的内存与多核模型，降低每包处理成本。**

![Linux协议栈与DPDK对比](Linux-netsatck-vs-dpdk.png)

图用于理解整体路径；DPDK 中的报文对象是 `mbuf`。以下以普通 socket 路径和 DPDK 持续轮询模式为主要比较对象。

## 一、DPDK 省掉了哪些开销？

| 开销 | 内核协议栈 | DPDK |
| --- | --- | --- |
| 中断与调度 | 中断触发 NAPI，通常通过软中断处理报文 | CPU 主动轮询，减少收包中断和睡眠唤醒开销 |
| 报文内存管理 | 管理 `sk_buff` 和接收缓冲区，也有缓存与复用机制 | `mempool + mbuf` 预分配、循环复用，配合 per-lcore cache 减少共享访问 |
| 用户态/内核态切换与拷贝 | 普通 socket I/O 经过系统调用，接收时通常复制到用户缓冲区 | 应用直接处理 DMA 写入的报文缓冲区，省去这段系统调用与拷贝 |

这里的“省掉”是减少常规路径开销，并非绝对没有调度、内存管理或数据复制。**内核纯转发本身也没有 socket 到用户态的拷贝。** [DPDK PMD 文档](https://doc.dpdk.org/guides-24.11/prog_guide/ethdev/ethdev.html)。

## 二、DPDK 从哪些角度优化？

| 角度 | 主要手段 | 核心思路 |
| --- | --- | --- |
| 并发与调度 | Poll mode、per-core/per-queue、lockless ring、RTC（同核完成收包、处理和发送提交） | 尽量不共享，减少锁竞争、跨核交接和调度开销 |
| 批处理 | Burst processing、SIMD、mempool bulk get/put | 一次处理更多数据，摊薄固定开销；SIMD 进一步提高并行计算效率 |
| 内存与缓存 | Hugepage、NUMA 感知、预分配、per-lcore cache、cacheline 对齐、prefetch | 减少地址翻译和远端访存，让数据更靠近 CPU、提前到达，并减少核心间干扰 |
| I/O 与硬件 | Kernel bypass（常见通过 VFIO/UIO）、少拷贝、RSS/Flow Director/checksum offload、平台 DDIO | 缩短软件路径，减少数据搬运，把适合的工作交给硬件 |

这些手段需要结合驱动和硬件使用；RSS、校验和卸载、DDIO 等也能用于内核路径，并非 DPDK 独有。

从个人角度来总结，DPDK其实是应用了一种编程模型，就是轮训模式代替原有的中断通知；把CPU的大部分时间放在轮训收包；同时应用了其他一种普适的优化方案比如批处理、无锁队列等来提高CPU利用率。

## 三、DPDK 编程模型

其实DPDK的编程模型也是和他的优化方式一样，比如：避免共享（使用per-cpu的变量）、使用预取、数据面路径简化（把旁路功能异步化）、数据面路径缩短（无状态使用RTC）、合理利用CPU结构（字节对齐避免false sharing）、必须共享时并发简化（比如无锁队列这种而不是使用大锁，使用rcu）、减少不必要的环节（比如使用mempool而不是使用时再分配内存）。。
后续准备写一篇专门的blog进行总结。