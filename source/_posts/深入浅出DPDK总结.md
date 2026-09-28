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

这些手段需要结合驱动和硬件使用；RSS、校验和卸载、DDIO 等也能用于内核路径，并非 DPDK 独有。[DPDK 性能编程指南](https://doc.dpdk.org/guides-24.11/prog_guide/writing_efficient_code.html)。

## 三、NAPI vs DPDK rx_burst

**NAPI 本身也会轮询、批量处理。主要区别在于谁驱动轮询，以及何时让出 CPU。**

| 维度 | 常规 Linux NAPI | DPDK 持续轮询 |
| --- | --- | --- |
| 触发方式 | 硬中断 → 调度 NAPI → poll，轮询期间通常屏蔽对应中断 | 应用循环调用 `rx_burst`，不依赖收包中断唤醒 |
| 批量上限 | 单次 poll 受 budget 限制，整体还受包数和时间预算限制 | 应用指定上限，如 32；返回实际数量，不等待凑满 |
| CPU 消耗 | 空闲时通常停止轮询，CPU 可处理其他工作或休眠 | 持续忙轮询通常占满一个逻辑 CPU，无包时也在空转 |
| 延迟 | 受中断合并、调度、排队和协议处理影响 | 减少中断路径，但仍受轮询间隔、业务处理和排队影响 |
| 退出与后续处理 | budget 用完但仍有工作时继续安排轮询；完成后通常恢复中断 | 本次调用达到上限或没有更多报文即返回；外层循环决定是否继续 |

不能统一写成“内核 μs 级、DPDK ns 级”，具体延迟取决于硬件、负载和测量范围。[Linux NAPI 文档](https://docs.kernel.org/networking/napi.html)。

**本质上，DPDK 以专用 CPU、预分配内存和更多应用侧管理责任，换取更高效率和更可控的处理路径。** 它提供高性能报文 I/O，应用需要的完整协议功能仍须另外实现。
